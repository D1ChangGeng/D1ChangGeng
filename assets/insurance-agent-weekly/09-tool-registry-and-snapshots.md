# 让模型自己选工具：Tool Registry、能力版本与运行快照

同一个知识助手会遇到三类问题：“这条免责规定是什么意思？”“这个产品的最低保费是多少？”“哪些产品具备指定功能？”第一类需要读取原文，第二类适合查询结构化字段，第三类还可能要求遍历目录和比较条件。

如果所有问题都固定先做向量检索，已有的业务接口就难以发挥价值。一个更通用的做法，是把可用能力描述给模型，让模型按当前任务选择动作。随之而来的工程问题是：这些工具怎样注册，哪些工具对本次运行可见，具体由哪个后端执行，以及执行期间配置变化会产生什么影响。

Tool Registry（工具注册系统）负责把这几个问题连接起来。它既是模型动作空间的来源，也是产品治理工具版本、可用状态和授权的边界。本文使用 Insurance Agent Runtime 的实际实现展开。

## 先区分四个容易混在一起的对象

在简单示例里，一个函数加上名称和描述就能成为工具。业务系统扩大后，需要把能力描述、使用范围和后端实现分开管理。

| 对象 | 中文含义 | 需要回答的问题 |
| --- | --- | --- |
| ToolSpec | 工具规格 | 能力叫什么，输入输出是什么，要求什么权限 |
| ToolProfile | 工具可见性配置 | 这类运行允许使用哪些能力 |
| BackendBindings | 后端实现绑定 | 当前能力实际由哪个实现提供 |
| ToolsetSnapshot | 工具集运行快照 | 这次 Run 使用的是哪一组具体定义和版本 |

这四者分别变化。检索能力可以保持相同输入输出，后端从合成实现切换为真实服务；同一个后端可以服务于多个 Profile；同一个 Profile 的新运行可以采用新的工具版本，已经开始的 Run 则保持自己明确的配置身份。[^registry]

```text
ToolSpec + ToolProfile + 当前主体
                    + BackendBindings
                           ↓
                  ToolsetSnapshot
                           ↓
                 SDK 函数工具集合
                           ↓
                   每次执行重新授权
```

这种拆分让“发现某项能力”“配置了后端”“已经准许执行”成为三个可以独立说明的状态。

## 工具规格描述业务能力

`stableToolSpecs` 从权威工具契约构造规格。以检索工具为例，关键字段可以整理成：[^catalog]

```ts
// 源码字段的阅读摘编，具体 Schema 来自项目契约。
{
  capabilityId: 'knowledge.search_documents',
  modelName: 'search_documents',
  version: '1.0.0',
  effect: 'read',
  requiredScopes: ['knowledge:read'],
  bindingKey: 'retrieval',
  handlerKey: 'searchDocuments',
  inputSchemaRef: '对应搜索输入的 Schema 引用',
  outputSchemaRef: '对应搜索输出的 Schema 引用',
  timeoutMs: 5000,
  maxOutputBytes: 200000,
}
```

`capabilityId` 是产品中的稳定能力名，`modelName` 是发给模型的函数工具名称。两者保持显式映射，例如 `knowledge.search_documents` 对应 `search_documents`。

`effect` 描述业务效果，当前运行工具支持只读能力。`requiredScopes`（所需权限范围）说明调用需要什么业务授权。`bindingKey` 指向服务端选择的能力实现，`handlerKey` 则关联本地处理入口。

输入输出 Schema（结构约束）是工具的真正协议。名称和描述帮助模型选择工具，Schema 约束实际参数和返回结构。版本字段用于记录这份协议和规格的身份，不能代替实际结构兼容性测试。

## 好的描述让模型理解动作边界

工具描述应该说明它能获得什么，以及返回结果在后续动作中如何使用。

搜索工具返回片段和可能可用的来源引用；来源工具根据精确 `sourceRef` 读取授权页；目录工具提供合法实体 ID；事实工具返回字段值、单位、有效期和来源。这样的分工可以直接对应用户需求。

输入中的 `content` 是公开动作说明，`query` 等字段是业务参数。例如：

```json
{
  "content": "查找示例保障甲等待期的正式条款",
  "query": "示例保障甲 等待期 适用条件",
  "companyIds": null,
  "productIds": null,
  "topK": 5
}
```

这是符合搜索输入字段形状的合成示例。模型表达查什么，服务端提供机构、主体、客户端和实际资源范围。`content` 用于记录模型声明的动作目的，执行是否成功则由 ToolInvocation（工具执行实例）与结果事实确认。

枚举产品时，模型应使用目录的分页信息；查精确金额时，需要读取字段和单位；查询优惠还需要有效日期。工具描述、字段设计和结果缺失语义，共同影响模型是否能够正确组合动作。

## Profile 决定允许选择的集合

当前源码定义了三个基础 Profile：`contract-test` 使用合成或录制数据，`advisor-synthetic` 使用合成业务，`advisor-readonly` 使用真实业务模式。它们记录能力 ID 集合和允许的数据模式。[^catalog]

模型是否真实调用，是另一条配置轴。一个真实模型可以使用合成工具测试动作选择；一个脚本化模型也可以驱动真实程序路径测试状态机制。仅从 Profile 名称推断全部运行性质，会丢失这些差异。

Profile 与当前主体共同决定可见工具。当前主体缺少某项业务 scope 时，目录会过滤该能力。具备 Runtime 操作权限，表示可以请求运行；具备业务读取权限，才允许相应工具查询业务数据。

这里有一个命名上的细节：对 Harness 发起执行属于 `runtime:write`，而执行中的工具可以仍是 `effect:read`。前者描述运行控制面的写操作，后者描述业务后端是否产生写效果。它们约束不同资源，因此并不冲突。

## 后端可用性需要有证据

Backend Descriptor（后端描述）包含后端身份、版本、数据模式、启用状态和就绪程度。实现中的 `ready()` 对真实模式要求 `live_verified`，对合成和录制模式要求 `schema_verified`。[^registry]

这两个标志分别表示通过真实接口验证与通过协议验证。它们的产生需要独立的验证过程，Registry 只是消费状态。填入一个服务地址或创建一份配置，本身不能证明接口的请求、响应和授权都已经可用。

`listTools()` 适合发现能力。它可以返回一项仍处于 `pending_resource` 的工具，使外部调用方知道该能力存在但资源尚待准备。`toolset()` 在创建实际运行快照时，则要求本次已选能力的后端满足准入条件。

所以目录读取与真实业务调用是两条不同路径。读取目录不应触发业务请求，也不应授予新的执行权限。

## 快照冻结的是身份，当前授权仍然生效

RunSnapshot（运行快照）记录本次运行的模型配置、工具集合、策略摘要和能力版本。工具集合内部还包含规格、Schema 摘要、后端描述、主体和限制参数。

```ts
// 摘编自 ToolRegistry.createSnapshot() 的核心返回结构。
return freezeJson({
  snapshotId: randomUUID(),
  profileId,
  modelProfileId: selectedModel.modelProfileId,
  sdkVersion: selectedModel.sdkVersion,
  modelConfig: selectedModel.modelConfig,
  toolset,
  toolsetHash: contentDigest(toolset),
  policyHash: contentDigest({
    owner: toolset.owner,
    scopes: toolset.scopes,
    policyVersion: this.policy.version,
  }),
  bindingVersions: Object.fromEntries(
    toolset.entries.map(entry => [
      entry.spec.capabilityId,
      entry.spec.version,
    ]),
  ),
});
```

`bindingVersions` 在该实现中保存能力规格版本，后端具体 revision（修订身份）还位于工具集合的后端描述里。读取快照时应检查完整结构，而不只看一个字段名称。

`bindingsFor()` 根据当前注册内容重建预期集合，并比较摘要、策略和版本。配置已经改变时，旧快照可以得到明确的不匹配结果。这比悄悄用新工具继续旧任务更容易解释，也为检查点恢复提供条件。

快照并不会永久保留执行权限。真正调用工具时，系统还会查询当前授权，并与请求或快照中的权限取交集。当前策略可以收紧权限，但不能借一次重新查询扩大原请求的能力范围。[^registry]

```text
本次可执行权限
  = 请求或运行快照允许的权限
    ∩ 当前主体仍然拥有的权限
    ∩ 工具要求及资源准入条件
```

这是逻辑表达，实际 scopes 支持通配范围，不能只用字符串相等计算交集。源码的 `permits()` 与 `intersectScopes()` 专门处理精确权限和范围权限的包含关系。

## 从快照到一次实际执行

`bindingsFor()` 返回 SDK 工具包装器需要的函数绑定。模型输出调用意图后，执行继续进入 Registry 的 `invoke()`：读取当前主体，验证业务 scope 和后端状态，解析去掉 `content` 的业务参数，再调用后端。[^registry]

返回结果还会进行三类检查：输出 Schema 是否正确，provenance（来源声明）是否匹配后端数据模式与版本，以及结果条目中的来源引用是否被外层引用集合覆盖。

因此，安全与一致性并不是只在“注册工具”时检查一次。注册决定合法结构，运行快照决定本次身份，调用边界决定当前能否执行，输出校验决定结果能否被接受。

## 一次配置变化会影响哪些地方

考虑把检索后端从版本 A 改为版本 B。若函数签名不变，模型仍然使用同一个 `search_documents` 工具名。但返回数据的语料、排序方式或缺失字段可能变化，后端 revision 和数据集版本就需要反映这种变化。

新的 Run 可以使用新快照，原来的 Run 则需要按旧快照和当前注册状态进行兼容检查。恢复时也要核对工具集合和模型配置。所谓“无缝替换”，应当是明确兼容契约后的结果，而不是忽略版本。

当前实现还在构造 Registry 时冻结描述和调用方法，使外部调用方之后修改原对象不会替换已经选择的执行闭包。这保护了服务启动后的配置身份。动态能力加载可以作为后续产品选择，但需要重新设计版本切换与活动运行规则。

## 怎样验证工具系统，而不是只验证工具函数

一项函数单元测试只能说明给定参数下返回什么。Registry 测试还需要经过发现、快照、绑定和当前授权。

例如，同一主体在两个 Profile 下看到不同集合；一个目录项存在但后端尚未就绪，创建执行快照应按规则处理；运行期间撤回业务 scope，下一次调用应受到限制；结果声明错误的数据模式或遗漏来源引用，输出检查应拒绝。

还可以保存一个快照后更改工具版本，再执行 `bindingsFor()` 或检查点恢复。测试应确认发生明确版本判断，而不是悄悄采用新的定义。

最后，用真实模型分别处理条款解释、精确字段和产品枚举，记录它选择了哪些工具。这样的实验观察描述和动作空间是否有用；确定性测试则保证它选择的每个动作都受到相同治理。

## 工具注册系统连接模型选择与业务实现

模型自主性来自可以选择的动作和可用反馈。Tool Registry 把动作空间定义清楚，又把每个动作连接到有版本、有权限和有结果契约的业务实现。

当一份用户问题最终变成某次工具调用时，我们已经能够解释：为什么这项能力可见，为什么当前允许执行，使用了哪个后端，以及结果为什么被接受。接下来，需要把这些组件真正装配进服务：[依赖装配与业务适配](10-composition-and-business-adapters.md)。

## 实现依据

[^registry]: Insurance Agent Runtime 源码快照 `c66112dfd8676b47b143dc2e26739278cf4b0dde`：`apps/runtime/src/registry/registry.ts`，重点为 `listTools`、`toolset`、`createSnapshot`、`bindingsFor`、`currentPrincipal` 和 `invoke`。
[^catalog]: 同一快照：`apps/runtime/src/registry/catalog.ts`、`registry/contracts.ts`，以及 `packages/contracts/src/tools.ts`。文中的六类能力、Profile 和参数形状来自这些文件。
[^sdk]: SDK 函数工具接入可参考 [OpenAI Agents SDK：Tools](https://openai.github.io/openai-agents-js/guides/tools/)；本工程额外实现的版本、快照和授权规则以 Registry 源码为准。
