# 从一次条款查询开始：把模型循环放进可运行的 Harness

一位顾问输入：“请查一下示例保障甲的等待期，并给出条款出处。”系统先查资料，再把检索结果交给模型，最后返回一段文字。第一次实现这个功能时，程序可能只有几十行。

接下来，顾问补充了一句：“意外伤害也适用吗？”与此同时，浏览器断开连接，检索服务响应变慢，另一个请求又使用相同的请求编号发了进来。现在需要回答的问题变成了：这两句话属于哪段对话，哪次执行仍在进行，哪些工具已经完成，模型下一次应该看到什么，以及外部系统怎样知道任务最终结束。

这些问题构成了 Harness 的工作范围。本文将 Harness 称为**智能体执行控制系统**：它接收任务，组织模型和工具运行，保存执行事实，并通过稳定接口输出状态与结果。Insurance Agent Runtime 是本文使用的工程实例；示例产品和咨询内容均为合成材料，读者也可以把它们替换成合同查询、技术支持或企业知识检索。

## 先看清楚最短的业务链

RAG（Retrieval-Augmented Generation，检索增强生成）把外部资料放入模型的输入，使回答能够参考具体证据。一个固定链路可以写成：

```text
用户问题
  → 检索相关片段
  → 将片段与问题组成模型输入
  → 调用模型生成答案
  → 返回答案和引用
```

这里的“固定”指调用顺序由程序预先确定。例如每个问题查询一次知识库，再调用一次模型。下面是用于说明边界的 TypeScript 接口示例；`retrieve` 和 `generate` 由应用注入实际实现。

```ts
type Evidence = {
  id: string;
  text: string;
  sourceRef: string | null;
};

type BaselineDependencies = {
  retrieve(question: string, signal: AbortSignal): Promise<Evidence[]>;
  generate(input: {
    question: string;
    evidence: Evidence[];
    signal: AbortSignal;
  }): Promise<string>;
};

async function answerOnce(
  question: string,
  dependencies: BaselineDependencies,
  signal: AbortSignal,
): Promise<{ answer: string; evidence: Evidence[] }> {
  if (!question.trim()) throw new Error('QUESTION_REQUIRED');
  signal.throwIfAborted();
  const evidence = await dependencies.retrieve(question, signal);
  signal.throwIfAborted();
  const answer = await dependencies.generate({ question, evidence, signal });
  return { answer, evidence };
}
```

这个函数适合观察业务输入输出。它已经把取消信号传入依赖，也保留了检索证据。但它的返回值还没有说明“这次工作属于哪个持久任务”。进程退出后，调用栈、局部变量和等待中的 Promise 都会消失。

当系统允许模型根据证据再次查询时，还会多出一层循环：模型看到工具定义，提出调用意图；程序执行工具，把结果交回模型；模型再选择继续查询或生成答复。OpenAI Agents SDK 提供了这种模型与工具交互的执行机制。产品侧仍然需要为这段循环提供任务身份、持久状态和执行边界。[^sdk]

## 一次请求有三种不同的身份

理解执行系统，可以先区分三个对象。

**Thread（对话线程）**保存一段持续对话的归属和历史。顾问今天查询一个产品，随后继续追问，交互可以留在同一个 Thread 中。

**Turn（用户回合）**代表一次正式受理的用户输入。“查询等待期”和“意外伤害也适用吗”分别产生两个 Turn。

**Run（执行尝试）**代表完成某个 Turn 的一次实际尝试。某次尝试被取消后重新执行，用户问题仍然是同一个 Turn，运行记录则使用新的 Run。

```text
Thread T
├── Turn U1：查询等待期
│   ├── Run R1：第一次尝试，中断
│   └── Run R2：恢复执行，完成
└── Turn U2：追问意外伤害
    └── Run R3：完成
```

再向内部展开，**ModelStep（模型调用步骤）**记录一次进入模型边界的调用尝试，**ToolInvocation（工具执行实例）**记录一个具体工具动作的执行。它们提供不同的观察尺度：模型响应可以包含多个工具调用，而工具可能在该模型响应已经返回之后才开始执行。[^impl]

这些身份的价值，在失败时尤其明显。系统可以说清楚“同一问题重试了两次”“第二次尝试复用了哪个工具结果”“最后一个答复属于哪个回合”。只保存一组聊天文本，很难回答这些问题。

## 沿着源码走一次完整请求

工程使用 Fastify 承载 HTTP 接口，PostgreSQL 保存业务状态，pg-boss 提供持久任务队列，SDK Runner 负责模型与工具循环。**RunnerFacade（运行门面）**把 SDK 接到产品的 Run 生命周期上。[^composition]

```text
HTTP 请求
  → createControlHttpApp：校验请求与身份
  → RuntimeStorage.submitTurn：事务受理
  → PostgreSQL：输入、Turn、Run、快照、事件、Job
  → RunWorker：领取任务并取得执行租约
  → RunnerFacade.execute：建立本次运行环境
  → SDK Runner：模型请求 ↔ 工具执行
  → RuntimeStorage.finishRun：提交任务终态
  → runtime_events：持久事件
  → HTTP 查询或 SSE 输出
```

这张图描述的是函数调用和数据流。实现中，API 与 Worker 可以位于同一个 Node.js 进程，模块边界依然清晰。是否拆成多个服务，是后续部署问题；请求如何受理、哪个对象可以写入状态、结果何时成立，是现在就需要确定的产品语义。

`submitTurn()` 在一个数据库事务里保存用户输入、Turn、排队中的 Run、运行快照、事件和队列任务。只有这些内容一起提交，接口才返回受理结果。Worker 随后通过 Run ID 领取任务，而不是沿用原 HTTP 请求的调用栈。

因此，HTTP 返回 `202 Accepted` 表示“系统已经持久受理了工作”。执行完成则由 Run 的终态表达。把这两个时刻分开，外部调用方就能够用同一个 Run ID 查询、取消或观察进展。

## 装配代码解释了系统真正的依赖

Composition Root（依赖装配入口）是创建组件并连接其依赖的位置。工程中的 `composeRuntime()` 先建立存储，再创建 Worker 和 HTTP 应用，最后启动后台组件。下面保留了它的核心结构，省略可选配置与清理分支：

```ts
// 源码摘编：用于阅读装配顺序，不是独立启动脚本。
const storage = await createRuntimeStorage(options.database, {
  migrate: options.migrate,
  authorizeClaim: options.authorizeClaim,
});

const worker = options.workerFactory?.({
  storage,
  modelProvider: options.modelProvider,
});

const app = await createControlHttpApp({
  storage,
  authenticator: options.authenticator,
  snapshots: options.snapshots,
  cursorSigner: options.cursorSigner,
  registry: options.registry,
  onClose: dispose,
});

await worker?.start();
```

从这段代码可以读出几个重要事实。

HTTP 应用依赖存储、身份认证器和运行快照解析器。它接受业务输入，却由服务端选择可信模型配置和工具后端。Worker 也依赖同一套产品存储，所以它能领取接口受理的任务，并写回同一份运行事实。

模型 Provider（模型提供方适配器）通过装配传入。工具背后的检索客户端、数据库客户端和凭证留在受控依赖中。模型得到的是工具说明、参数结构和工具返回内容；这些内容通过明确的数据通道进入模型请求。

关闭顺序同样属于设计。实际 `dispose()` 先停止 Worker，再停止维护任务，最后关闭存储。这样，仍在收尾的执行器可以先处理自己的状态，而不是在数据库已经关闭之后尝试提交终态。[^composition]

## SDK 与 Harness 分别推进什么

SDK Runner 处理模型调用和工具循环。模型返回工具意图后，SDK 调用对应函数，将工具结果加入后续输入；满足其结束条件后，运行返回最终输出。[^sdk]

Harness 则围绕这段循环维护产品事实：当前运行是谁的，是否仍有执行权，预算是否允许下一次调用，工具结果是否已经持久保存，以及最终内容能否提交到当前回合。

工程中的 `RunnerFacade.execute()` 会构造 `RunEnvironment`（单次执行环境），其中包含当前主体、运行快照、存储、取消信号、预算和租约。它随后创建 SDK Agent、持久 Session 和工具包装器，并调用 `runner.run()`。[^runner]

最容易误读的一处是 SDK 事件流。门面中确实遍历了 SDK stream，但权威状态记录主要发生在模型、工具和 Session 的可等待边界上。模型调用前创建 ModelStep，响应完成时提交输出条目；工具返回后提交 ToolInvocation 及结果。单纯在事件循环里“看到一个事件”，并不自动等于产品状态已经提交。

这样设计的原因很具体：只有在工具结果真正保存之后，SDK 才能继续发起依赖该结果的模型请求。持久化是执行链上的约束，而不是旁边的一份调试日志。

## 先完成一条纵向链，再扩展工具数量

Vertical Slice（纵向切片）指一次贯穿多层系统的小功能。对这个 Harness 来说，一个有价值的切片是：

```text
提交一条输入
  → Worker 实际领取
  → 受控模型提出一个工具调用
  → 工具返回合成证据
  → 结果实际写入数据库
  → 第二次模型调用生成答复
  → Run 完成
  → 外部读到完整事件
```

它比单独证明“接口能返回 JSON”或“模型能生成一段文字”多验证了组件之间的连接。契约、事务、队列、模型边界、工具边界和终态都被真正经过了一次。

这里还要区分两种常见测试模式。默认合成启动模式可以用固定模型输出验证服务生命周期；一个能列出六项工具的目录，说明工具配置已经装配。要证明自动工具循环，需要使用会输出调用意图的受控模型，或已经获得授权的真实模型。工程中的 Runner 与控制面集成测试承担了前一种验证。[^tests]

一个受控案例的预期关系可以表达为：

```text
一个已受理 Turn
一个当前 Run
两次 ModelStep
一次 ToolInvocation
一个最终内容引用
一组按 Thread 排序的持久事件
```

这些数字来自人为设计的测试剧本，作用是检查协议。真实模型可能一次直接回答，也可能进行更多合理查询，行为效果应另行观察。

## 固定 RAG 依然是有价值的基线

Baseline（比较基线）让我们知道新增自主能力带来了什么变化。可以用相同的问题、相同的检索后端和相同模型，对比固定一次检索与允许补查的系统。需要记录实际查询、返回证据、模型调用数、耗时，以及回答中哪些要点已经有来源支持。

例如，固定检索找到了“等待期为若干天”，补查又找到了意外情形的适用条件。新增的一次调用带来了一个具体证据要件，而不是仅仅让回答更长。反过来，如果第二次查询重复返回相同片段，轨迹也能让我们看见额外成本。

Harness 在这里提供的是可信测量基础：每次模型调用和工具执行都有身份，实际参数和结果可以回溯，用量与终态能够关联。系统效果的判断再建立在这些事实之上。

## 把系统边界落到可以观察的事实

一次保险咨询最终会变成一段自然语言答复，但工程真正交付的是支撑这段答复的完整运行系统。请求入口负责受理，Worker 持有执行资格，模型与工具边界保存事实，Session 构造下一次输入，存储提交终态，事件接口对外表达已经发生的变化。

读到这里，再看“用户发来一句话”就会有更具体的理解：它首先成为一个可以持久追踪的执行请求，然后才进入模型循环。接下来最值得确定的是这些对象的契约——哪些字段定义身份，哪些字段保存事实，哪些指标能够证明系统确实工作。

延伸阅读：[契约、运行实体与证据标尺](02-contracts-and-runtime-facts.md)。

## 实现依据

[^impl]: Insurance Agent Runtime 源码快照 `c66112dfd8676b47b143dc2e26739278cf4b0dde`：`apps/runtime/src/storage/store.ts` 的 `submitTurn`、`startModelStep`、`startInvocation`、`finishRun`。本文引用的是该快照的行为；教学请求与轨迹使用合成内容。
[^composition]: 同一源码快照：`apps/runtime/src/composition/runtime.ts`，`composeRuntime` 与 `dispose`。物理装配和关闭顺序以这两个函数为依据。
[^runner]: 同一源码快照：`apps/runtime/src/runner/facade.ts`，`RunnerFacade.execute`。该实现使用项目锁定的 Agents SDK 0.18.0；SDK 自身的术语与产品实体分别映射。
[^tests]: `tests/control/http.test.ts` 与 `docs/implementation/runner-preparation.md` 记录的队列、真实 PostgreSQL、受控 SDK 工具链测试。这里说明既有测试的验证对象，不表示本文重新执行了完整工程回归。
[^sdk]: [OpenAI Agents SDK：Running agents](https://openai.github.io/openai-agents-js/guides/running-agents/)。用于理解 SDK 的循环职责，具体集成行为以本文绑定的项目版本为准。
