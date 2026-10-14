# 一次工具调用如何成为可靠执行：ToolInvocation 的完整生命周期

用户要求比较两款产品的保障责任与缴费条件。模型在一次响应中提出两个查询：一个检索条款，一个读取结构化产品字段。两个外部接口返回速度不同，其中一个结果还包含缺失字段。

从模型视角看，这是两个工具调用。从工程视角看，它们是两个需要独立管理身份、权限、状态和结果的执行实例。Harness（智能体执行控制系统）要明确：哪个模型响应提出了调用，哪个后端处理了参数，哪份结果已经保存，以及下一次模型调用何时可以使用结果。

Insurance Agent Runtime 使用 ToolInvocation（工具执行实例）承载这些事实。本篇沿 `createRunTools()` 展开，从模型调用意图一直走到结果持久化与模型继续执行。

## ToolCall 表示意图，ToolInvocation 表示执行

ToolCall（工具调用意图）是模型输出中的结构化条目，包含工具名称、参数和调用标识。ToolInvocation 则是运行系统创建的执行记录。两者通过 `providerCallId`、`callItemRef` 和 `modelStepId` 关联。[^tools]

```text
ModelStep S1
├── ToolCall C1：查询产品甲条款
└── ToolCall C2：读取产品乙字段

ToolInvocation V1
  modelStepId = S1
  providerCallId = C1 的 callId
  callItemRef = C1 的条目 ID

ToolInvocation V2
  modelStepId = S1
  providerCallId = C2 的 callId
  callItemRef = C2 的条目 ID
```

一个 ModelStep（模型调用步骤）可以提出多个意图，执行实例分别记录成功、失败和取消。模型响应完成时，ModelStep 就可以结束；后端工具仍然继续自己的生命周期。

在模型输出被保存时，`RecordedModel.complete()` 会建立 `callId → ModelStep 与调用条目` 的映射。后续工具包装器据此检查 SDK 要执行的调用是否确实来自当前运行已记录的模型输出。工具名和调用身份都匹配，才能继续。[^model]

## 工具定义如何变成 SDK 可执行函数

ToolRegistry（工具注册系统）根据运行快照生成一组 `ToolBinding`（工具执行绑定）。绑定包含能力标识、模型工具名、版本、参数 Schema（结构约束）、权限要求、数据模式和实际执行函数。

`createRunTools()` 再把这些绑定转换成 SDK 的函数工具。除了使用 SDK 完成参数解析，包装器还接入产品的执行治理。

```ts
// 源码摘编：完整工具定义还包括结果提取与错误处理。
const boundTool = tool({
  name: binding.modelName,
  description: binding.description,
  parameters: binding.parameters,
  outputSchema: z.object({}).catchall(z.unknown()),
  errorFunction: null,
  execute: async (input, context, details) => {
    // 校验当前 Run、工具调用身份、权限和预算。
    // 保存执行实例，调用 binding.execute，再返回验证后的结果。
  },
});
```

这里 SDK 外层的宽对象 Schema 并不代表业务结果任意放行。执行函数内部仍调用 `binding.outputSchema.parse(result)`，Registry 也会验证具体能力的输出结构。SDK 包装层负责接入执行协议，严格业务契约在产品边界继续成立。[^tools][^registry]

这是一种适配取舍：框架的错误转译和结果提取行为，需要通过锁定版本的实际测试确认。把输入错误自动转换成一段没有产品执行身份的结果，可能破坏后续追踪；因此实现显式处理这类行为。

## 执行前先固定调用身份和业务参数

工具开始执行时，包装器先检查 SDK 上下文中的 Run ID，再从 `details.toolCall` 读取调用标识，并与模型边界保存的事实匹配。恢复时已经完成的调用，也通过已保存集合进行检查。

接着处理权限和参数。工具输入中的 `content` 是模型生成的公开动作说明，例如“分别查询两款产品的缴费条件”。业务后端只接收剩余业务参数：

```ts
const { content, ...businessArguments } = parameters;

await storage.startInvocation(lease, {
  invocationId,
  modelStepId: call.stepId,
  providerCallId: callId,
  capabilityId: binding.capabilityId,
  version: binding.version,
  content,
  arguments: businessArguments,
  dataMode: binding.dataMode,
  callItem: runtimeItem(call.canonical, call.itemId, call.stepId),
});
```

这段动作说明可以帮助外部读者理解“模型声明自己要做什么”，但它不是对模型内部推理的完整记录，也不是执行已经成功的证明。真正结果由调用状态、后端返回和持久事实确认。

机构、主体、上游地址和客户端凭证由本地环境提供。模型负责语义查询与业务筛选，服务端负责把筛选解释为当前授权范围内的实际请求。

## 结果返回后，还有两次重要检查

第一类检查针对结果结构。返回内容必须通过对应能力的输出 Schema，且序列化后的大小符合结果预算。`sourceRefs`（来源引用集合）也要能够从结果中提取并与具体数据关联。

第二类检查针对结果来源。Registry 比较输出的 `provenance`（数据来源声明）与当前 Backend Descriptor（后端描述）是否一致，包括数据模式、后端身份和数据集版本。结果里出现的来源引用还必须被总引用集合覆盖。[^registry]

例如，录制后端不能把返回值标为实时数据，某个字段条目也不能悄悄携带外层未声明的来源引用。类型正确只能证明结构可解析；来源一致性检查进一步确定“这份数据是谁提供的”。

对于过大的结果，Registry 的结果收敛函数按能力规则限制内容并保存对应缺口信息。执行包装器随后再检查最终字节大小。系统既需要保护资源，也需要让模型知道返回结果的覆盖范围。

## 最关键的顺序：提交结果，再继续模型

业务后端返回一个对象时，它还只是进程内的值。SDK 需要把它转换为自己的工具结果条目，而产品需要保存这个确切条目，才能在恢复和下一次模型请求中引用它。

工程先把验证后的结果放入本次调用的暂存映射，随后在 `customDataExtractor`（SDK 结果条目提取回调）取得 `rawItem`，为它分配产品条目 ID，并等待持久化完成。[^tools]

```ts
// 摘编自 customDataExtractor；省略失败分支。
const call = pending.get(toolCall.callId);
const id = randomUUID();
const item = {
  ...runtimeItem(rawItem, id, call.stepId),
  sourceRefs: [...call.sourceRefs],
};

await storage.finishInvocation(lease, call.invocationId, {
  status: 'succeeded',
  resultItem: item,
  sourceRefs: call.sourceRefs,
  checkpoint: true,
});

recordFact(facts, rawItem, id);
pending.delete(toolCall.callId);
```

`finishInvocation()` 在数据库事务中保存结果条目、执行实例终态和 `tool_invocation.completed` 事件；指定检查点时，还保存工具结果恢复边界。只有这些写入完成，回调才返回，SDK 才能继续使用该结果。

这建立了一条 Happens-before（先后依赖）关系：

```text
工具结果持久提交
  happens-before
下一次模型请求消费该结果
```

如果改成后台异步写日志再立即返回，模型可能已经依据某份证据给出结论，数据库却永远没有保存这份证据。进程一旦中断，系统就无法重建那次回答的真实输入。

## 工具状态与业务结果状态要分别阅读

执行实例的 `succeeded` 表示执行边界成功得到并提交了合法结果。业务结果中的 `status` 则可能是 `ok`、`partial`、`empty` 或某种结构化错误状态。

例如查询优惠得到空集合，外部请求和结果提交都可能成功，但业务含义是本次范围没有命中。读取结构化字段得到 `null` 和缺失说明，也可以是一次成功完成的工具动作。

反过来，参数非法、权限在执行期间撤回、调用超时或结果结构无法解析，会触发相应执行失败路径。有些拒绝发生在创建 ToolInvocation 之前，因此不能假定每一个模型意图都已经成为执行实例。[^tools][^registry]

| 层次 | 示例判断 | 回答的问题 |
| --- | --- | --- |
| 模型意图 | 工具名、参数和 callId | 模型提出了什么动作 |
| 执行实例 | running、succeeded、failed、cancelled | 系统如何执行并提交动作 |
| 业务结果 | empty、partial、missing | 这次查询获得了什么业务信息 |
| 来源状态 | 版本、授权、页码映射 | 信息能否按声明被核对 |

把四层合并为一个布尔 `success`，会丢失排障所需的大部分信息。

## 多工具并发改变时序，不改变身份

模型在同一响应中返回两个调用，SDK 可以根据 `maxFunctionToolConcurrency` 控制并发数量。工具总次数预算则由当前 Run 的共享计数限制。

假设调用 C1 花两秒，C2 花半秒。数据库事件可能先出现 V2 完成，再出现 V1 完成。这完全合理。调用发起顺序、完成顺序和后续模型输入的排列需要分别理解，通过稳定 ID 关联，而不是依据“列表第二项应该对应第二个返回”来猜测。

还要区分模型步骤与工具步骤。两个并发工具可以来自同一 ModelStep；真正的新模型请求开始时才产生下一个 ModelStep。工具数量增加不应该自动变成模型调用次数增加。

对于存在数据依赖的多跳任务，模型往往需要先得到前一个工具结果，再生成后一个调用参数。这类依赖通过后续模型步骤表达。调度器可以并发独立调用，但不能凭借并发设置消除业务数据依赖。

## 结果写入失败是最有价值的测试之一

一个可重复的测试剧本可以这样安排：模型第一次返回调用，合成工具成功返回，存储在提交工具结果时抛错。预期是下一次模型调用没有发生，执行实例进入相应失败收尾，原工具结果不会被伪装成已持久事实。

实际包装器在提交失败时尝试将 Invocation 标为 `outcome_unknown`（结果不确定），并向上抛错。如果数据库连这次收尾都无法完成，运行监督还需要在恢复后继续收敛状态。因此，“尝试标记未知”与“未知状态已经写入数据库”也要由读回证据区分。[^tools]

另一组测试应设置不同工具并发上限，观察实际峰值；再让输出超大、Schema 不匹配或来源集合缺项，检查失败落在哪个边界。工程的 Runner 和 Registry 测试说明记录了这些验证方向。[^tests]

## 复杂任务的可靠性建立在每个小动作上

跨产品对比看起来是一个大任务，底层却是多个有身份的调用和结果。每个调用都经过授权、预算、输出校验和持久化，后续模型才能在确定的事实基础上继续。

ToolInvocation 的意义就在这里：它把“模型说要查”与“系统已经完成并保存查询”连接起来。下一篇继续沿结果中的来源引用深入，看看这些数据怎样一直追溯到原始文档：[证据与授权来源](08-evidence-and-source-references.md)。

## 实现依据

[^tools]: Insurance Agent Runtime 源码快照 `c66112dfd8676b47b143dc2e26739278cf4b0dde`：`apps/runtime/src/runner/tools.ts`，`createRunTools` 与 `customDataExtractor`；`storage/store.ts` 的 `startInvocation`、`finishInvocation`。
[^model]: 同一快照：`apps/runtime/src/runner/model.ts`，`RecordedModel.complete` 保存调用条目并建立 `callSteps` 映射。
[^registry]: 同一快照：`apps/runtime/src/registry/registry.ts` 的 `bindingsFor`、`execute`、`invoke`，以及 `registry/output.ts`；业务结果结构见 `packages/contracts/src/tools.ts`。
[^tests]: `docs/implementation/runner-preparation.md` 与 `tests/registry/registry.test.ts` 的既有测试定位。SDK 工具接口另见 [OpenAI Agents SDK：Tools](https://openai.github.io/openai-agents-js/guides/tools/)，具体回调行为以项目锁定版本为准。
