# 把工具接到真实业务系统：依赖装配、Port、Adapter 与数据模式

模型已经能够调用 `search_documents`，合成测试也能返回条款片段。下一步是连接真实检索服务、产品目录和来源页。看起来只需要把一个函数里的假数据换成 HTTP 请求，实际集成却会遇到产品 ID、分页、单位、有效期、权限和版本等问题。

这说明工具接入存在两种兼容性：函数签名兼容，以及业务含义兼容。一个返回 `value: 10000` 的接口，仍然需要解释币种、缴费周期和来源。能够解析 JSON，只是集成的一个阶段。

Insurance Agent Runtime 通过 Composition Root（依赖装配入口）、Port（业务接口契约）和 Adapter（具体适配实现）连接这些系统。本文从一个检索工具的调用路径展开，说明如何让合成、录制和真实后端使用同一套运行机制。

## 从业务参数到本地依赖

模型看到的工具输入包含语义参数。例如查询内容、产品 ID 和召回数量。业务接口还需要主体、权限范围、取消信号和后端版本，这些信息来自 Harness（智能体执行控制系统）的可信运行环境。

源码用 `BusinessInput` 去掉工具输入里的公开动作说明 `content`，再通过独立的 `BusinessContext`（业务执行上下文）传入本地条件。[^ports]

```ts
export type BusinessInput<K extends ToolCapabilityId> =
  Omit<ToolInput<K>, 'content'>;

export interface BusinessContext {
  readonly principal: Principal;
  readonly signal: AbortSignal;
  readonly runId?: string;
  readonly snapshotId?: string;
  readonly backendRevision: string;
  readonly dataMode: DataMode;
}

export interface RetrievalPort {
  searchDocuments(
    input: BusinessInput<'knowledge.search_documents'>,
    context: BusinessContext,
  ): Promise<BusinessOutput<'knowledge.search_documents'>>;
}
```

这段代码表达的边界很直接：调用方提供“查什么”，本地环境确定“以谁的身份、在什么后端和模式下查询”。模型无需持有数据库连接或服务凭证，也不能靠工具参数替换当前机构。

Port 是类型契约。它没有独立运行的魔法，也不会自动生成网络服务。实现这个接口的对象，才负责完成实际查询。

## Adapter 把上游协议转成稳定能力

Retrieval Adapter（检索适配器）把产品请求转换成上游检索参数，再将上游返回归一成 Evidence（证据）结果。Source Adapter（来源适配器）处理文档定位和解析页。Catalog Adapter（目录适配器）处理保司与产品列表，包括分页和身份映射。

实际源码按如下目录组织：[^adapters]

```text
apps/runtime/src/business/
├── ports.ts
├── synthetic.ts
├── recorded.ts
├── adapters/
│   ├── retrieval.ts
│   ├── catalog.ts
│   ├── source.ts
│   ├── parsed-source.ts
│   └── pg-source-references.ts
└── composition/
    ├── configured-business.ts
    └── python-read-bridge.ts
```

一个 Adapter 可以直接调用 HTTP，也可以使用已有 SDK 或局部连接桥。是否增加 Service（业务服务层），取决于是否存在需要集中维护的规则，例如授权、来源目录、缓存和数据口径。目录层级本身不提供可靠性，稳定的输入输出和实际执行规则才提供。

例如来源读取不仅是下载 JSON。它先解析授权来源引用，检查文档版本和页码，再从解析结果取出指定页，返回前复核当前条件。这些逻辑应由本地业务边界统一处理，而不是散落在不同工具函数中。

## 绑定方法时捕获真实实现

`bindBusinessPorts()` 将业务 Port 转换为 Registry（工具注册系统）可使用的 BackendBinding（后端绑定）。函数先捕获各接口方法，再建立能力 ID 到方法的显式分派。[^bindings]

```ts
// 源码摘编。
const search = ports.retrieval.searchDocuments.bind(ports.retrieval);
const source = ports.source.getSourcePage.bind(ports.source);
const products = ports.catalog.listProducts.bind(ports.catalog);

const dispatch = {
  'knowledge.search_documents': (input, context) =>
    search(input, context),
  'knowledge.get_source_page': (input, context) =>
    source(input, context),
  'catalog.list_products': (input, context) =>
    products(input, context),
};
```

完整实现还解析并冻结后端描述，返回包含 set ID、修订身份和绑定映射的对象。后续即使原调用方修改了 `ports` 对象的方法，也不会替换已经捕获的执行入口。

客户端和凭证保留在这些本地闭包里。RunSnapshot（运行快照）保存可比较的配置身份和版本，而不是序列化数据库连接、函数或秘密值。

实际调用链因此是：

```text
SDK 函数工具
  → ToolInvocation 执行包装器
  → Registry 当前授权与结果校验
  → BackendBinding.invoke
  → Port 的具体实现
  → 上游 HTTP / SDK
```

每一层都有自己的职责。调用多几层函数的目的，是让业务协议、权限和执行事实有明确归属，而不是为了把简单函数包装得复杂。

## 数据模式与模型模式是两条轴

Synthetic（合成）、Recorded（录制回放）和 Real（真实）描述的是业务数据来源。模型执行则可以使用 Scripted Model（脚本化模型替身）或真实模型。

| 模型 | 业务后端 | 适合验证 |
| --- | --- | --- |
| 脚本化模型 | 合成或录制 | 生命周期、调用关联、故障和事件协议 |
| 真实模型 | 合成 | 工具选择和多轮动作行为 |
| 真实模型 | 已授权真实接口 | 实际查询链路与业务证据获取 |

这张表避免了一种常见混淆：在测试中运行真实的 HTTP、数据库和 SDK，并不意味着返回内容来自真实业务；使用真实模型，也不自动证明条款答案经过人工核验。

工程把数据模式写入后端描述和 ToolResult（工具结果）的 provenance（来源声明）。Registry 还会核对结果声明是否与后端身份一致。录制数据因此不会在模型输入里自然变成“刚刚从真实服务查询”的结果。[^modes]

## 合成数据应该保留业务难点

好的 Fixture（测试夹具）不只是几条永远成功的数据。它应该覆盖具有不同含义的返回：合法空列表、字段未知、部分分页、过期优惠、来源尚未核验、输出结构错误，以及权限变化。

例如结构化产品字段可以返回一个明确未知值，附带缺失说明和来源范围。优惠查询需要指定 `asOf`（查询时点），并返回有效期。测试数据可以使用示例产品，但单位、日期和缺失语义仍需与真实契约一致。

这样，当真实资源尚在准备时，Harness 主链依然能验证：模型看到空结果后如何行动，工具错误是否具有稳定分类，部分结果能否正确持久化，以及最终事件怎样表达状态。

录制模式则适合保存已经获得授权的响应形状，用于格式回归。它需要绑定请求、主体、后端和数据集身份，确保回放的是当前测试所声明的情境。录制响应不能代替接口在今天仍然可用的实际读回。

## 真实接入需要验证业务含义

把一个服务从 configured（已配置）推进到 live verified（真实验证通过），至少需要确认操作范围和返回含义。这里的 Readback（实际读回核对）就是使用真实请求检查接口行为，再保存对应证据。

目录接口尤其容易暴露问题。第一页面有数据，并不说明已经完成全量读取；同名产品也不一定属于同一实体。Adapter 应核对分页规则、稳定 ID、公司归属、重复项和覆盖范围。

检索接口需要确认查询参数、过滤语法、向量语义和排序。`score`（分数）必须与 `scoreMetric`（度量方式）一起理解，不能把不同后端的数值直接按相同阈值比较。

事实和优惠接口还需要业务口径：金额对应哪种币种，是首期还是年度保费；有效期采用什么时区或日期边界；未知值和不适用分别怎样表示；来源文档与页面是否可以访问。[^integration]

因此，契约测试检查的不只是字段存在，还包括这些字段在业务上的解释。

## 授权在慢初始化之后仍需复核

真实客户端可能在第一次调用时才读取本地配置或建立连接。这个初始化过程具有异步等待。若权限在等待期间发生变化，入口时的检查就不足以说明实际发送时仍然允许执行。

工程记录过相应修复：在首次私有客户端查找结束后，再读取当前授权，随后才允许发起模型请求。工具资源和来源读取也在自己的实际请求路径上进行复核。[^integration]

这体现了 Time-of-check / Time-of-use（检查时点与使用时点）之间的差异。解决方法是把有意义的当前条件检查放到实际动作附近，并让拒绝结果进入正常执行状态处理。

配置摘要可以帮助比较资源和版本，但摘要不是授权凭证。一个非空字符串也不是实际读回证明。验证报告和当前授权由各自受控来源提供，装配代码消费这些输入。

## 用同一条纵向测试验证真实分支

业务适配的验证可以从最窄路径开始。用一个本地测试 HTTP 服务返回符合真实协议的合成响应，产品沿真实 Adapter 解析，然后经过 Registry、工具包装器和持久存储完成一次循环。

这条测试真正经过了“真实分支”的代码，但数据仍然是合成。记录时应准确表述为协议和装配验证。随后，在授权环境中做少量实际接口读回，确认请求、响应、版本和权限与测试假设相符。

这种分层可以减少反复调用昂贵外部服务，又保留接口变动的检测能力。固定测试负责每日回归，真实读回负责确认外部世界仍然满足契约。

## 服务启动与关闭也属于装配

`composeRuntime()` 先创建产品存储，再装配 Worker、HTTP 和维护任务。HTTP 入口引用快照解析器和 Registry，Worker 使用同一存储领取 Run。关闭时先停止执行和维护组件，再释放存储。[^composition]

```text
启动
  数据库与配置 → 业务绑定 → HTTP / Worker → 服务就绪

关闭
  停止接入与观察 → 执行器收尾 → 维护任务停止 → 存储关闭
```

具体关闭钩子还处理 SSE（服务器发送事件）观察连接。观察者拥有自己的生命周期，后台 Run 的取消则由执行控制面管理。部署可以先采用单进程，再按负载拆分，但这些依赖关系应一直保持明确。

## 稳定契约让业务能力可以持续替换

一个可维护的 Agent 系统，需要让模型工具名称相对稳定，让业务后端可以按版本演进。Port 约定能力，Adapter 转换协议，Composition Root 选择依赖，Registry 冻结运行身份并执行治理。

当这些边界清晰时，真实集成就不再是“替换假数据”的一次性动作，而是一套可以重复验证的接入过程。工具结果成为持久事实后，还需要以稳定协议对外输出：[持久事件与流式传输](11-durable-events-and-streaming.md)。

## 实现依据

[^ports]: Insurance Agent Runtime 源码快照 `c66112dfd8676b47b143dc2e26739278cf4b0dde`：`apps/runtime/src/business/ports.ts`。代码片段保留实际类型关系。
[^adapters]: 同一快照的 `apps/runtime/src/business/adapters/` 与 `business/composition/` 实际目录。本文使用工程实现路径，而不是早期设计包中的建议布局。
[^bindings]: 同一快照：`apps/runtime/src/registry/backends.ts`，`bindBusinessPorts`。
[^modes]: 同一快照：`business/synthetic.ts`、`business/recorded.ts`、`registry/registry.ts`，以及 `packages/contracts/src/tools.ts` 的 provenance 定义。
[^integration]: `docs/implementation/real-business-composition-preparation.md`、`facts-promotions-handoff.md` 和 `architecture-decisions.md` 保存的接口口径、当前授权修复和接入规则。它们包含历史阶段记录，真实支持范围仍按对应资源证据读取。
[^composition]: 同一快照：`apps/runtime/src/composition/runtime.ts`、`transport/http/app.ts` 和 `runner/worker.ts` 的组件生命周期。
