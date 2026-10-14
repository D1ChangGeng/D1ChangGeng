# 从排队任务到模型循环：Worker、执行租约与 RunnerFacade

接口已经返回 Run ID，但模型还没有开始生成。数据库里的运行状态是 `queued`，队列中有一个携带 Run ID 的 Job。接下来，谁来把这份工作真正执行起来？

对于需要调用模型和外部工具的服务，执行可能持续数秒到数分钟。HTTP 连接可以断开，进程可以重启，队列也可能再次交付任务。运行系统需要给每个后台执行者一个明确资格，并且在写入状态时检查这个资格仍然有效。

Insurance Agent Runtime 把这段职责分给 RunWorker（后台任务执行器）、执行租约和 RunnerFacade（运行门面）。本篇追踪一条“查询合成产品条款”的任务，直到它完成模型、工具和再次模型调用的循环。

## 队列交付任务，存储决定谁可以执行

pg-boss 是基于 PostgreSQL 的任务队列。产品受理请求时已经保存 Job，Worker 读取 Job 数据后取得 `runId`。此时它得到的是“有一份任务需要处理”的通知，还需要调用产品存储中的 `claimRun()` 领取执行资格。[^worker]

```ts
// RunWorker.start() 的核心调用顺序，省略失败收尾。
const data = runJob.parse(job.data);
const claimed = await this.storage.claimRun(
  data.runId,
  this.ownerId,
  this.leaseMs,
);

if (!claimed) continue;
await this.facade.execute(claimed);
```

`claimRun()` 检查 Run 的状态、是否已有租约、是否已经开始。成功时，它把 Run 改为运行中，建立 Lease（有期限的执行资格），记录 `run.started` 事件，并返回输入所需的配置快照和已确认主体。[^lease]

把队列交付与产品领取分开，可以让重复交付经过同一套状态检查。任务队列负责送达，运行系统负责决定该任务此刻是否可以再次产生执行效果。

## 租约为什么要带 owner、fence 和过期时间

租约记录包含运行身份、持有者、代次标记和过期时间。Fence（隔离令牌）用来识别某次领取的写入资格，Owner ID（持有者标识）识别当前 Worker。

```text
runId      ：正在执行哪一次 Run
ownerId    ：哪个 Worker 持有执行资格
fence      ：这份资格的代次标记
expiresAt  ：数据库认定的有效期
```

Worker 通过 heartbeat（心跳续租）延长有效期。写入模型步骤、工具结果和运行终态时，存储层统一经过 `withLease()`。它在事务中锁定运行所属线程，检查令牌和有效期，执行动作后再次检查。有效期判断使用数据库时钟。[^lease]

为什么写完还要检查一次？因为一次事务可能在异步工作中停留较久。入口时有效的租约，在最后提交前可能已经过期。检查前后条件，可以把“曾经有权执行”和“此刻可以提交结果”区分开。

本实现的过期恢复策略也很明确：监督逻辑将失去租约的运行收敛为中断或相应取消终态，随后通过显式重试或检查点恢复创建新的 Run。它没有把进程重启自动解释为同一个运行从任意位置继续。

因此，租约不是持久程序计数器。它管理写入资格；继续执行的具体位置由检查点和历史事实决定。

## RunnerFacade 把 SDK 放进产品生命周期

RunnerFacade 是一个薄门面：它接收已经领取的 `ClaimedRun`，创建本次运行的依赖环境，再调用 SDK。工程中的核心入口是 `RunnerFacade.execute()`。[^facade]

这个方法首先为当前 Run 建立 AbortController（取消控制器），解析执行预算，启动总时限计时器，再安排心跳。心跳除了续租，还读取持久的 `cancel_requested` 状态，把用户取消命令传递到本地执行信号。

随后建立 RunEnvironment（单次运行环境）：

```ts
// 源码摘编：这些对象留在服务端执行环境中。
environment = {
  claimed,
  principal: claimed.principal,
  snapshot: claimed.snapshot,
  storage: this.storage,
  signal: controller.signal,
  budget,
  lease: claimed.lease,
};
```

Principal（主体）确定是谁在运行，Snapshot（配置快照）确定运行采用什么模型和工具版本，Storage（存储）提供事实提交能力，Signal（中断信号）传递取消，Lease（租约）控制写入资格。

这些本地对象与模型上下文有清晰边界。源码传给 SDK `context` 的核心值是 `{ runId }`，完整环境通过受控闭包和依赖传递使用。模型实际看到的内容来自指令、消息历史、工具定义和工具结果。

## 一次模型请求怎样成为 ModelStep

ModelStep（模型调用步骤）记录一次进入产品模型边界的尝试。工程使用 `RecordedModel` 包装具体 Provider（模型适配实现）。它保留 SDK 的 Model 接口，同时增加产品事实记录。[^model]

`begin()` 先构造模型可见输入，检查上下文预算和调用预算，再执行当前授权回调。确认运行仍可继续后，持久化 ModelStep。

```ts
// 摘编自 RecordedModel.begin()。
const projected = this.projectRequest?.(request) ?? request;
assertModelInputBudget(
  projected,
  contextBudgetFor(this.environment.snapshot.modelConfig),
);
await this.beforeModelCall?.(this.environment, projected);
this.environment.signal.throwIfAborted();

const stepId = randomUUID();
await this.environment.storage.startModelStep(
  this.environment.lease,
  { stepId, ordinal: ++this.ordinal },
);
```

实际实现还在异步授权前后检查模型调用上限。调用次数预算必须和实际发送路径一致，否则一次慢授权或并发路径就可能绕过预期限制。

对于流式模型，适配层提供规范化事件。这里的 `output_text_delta` 表示文本增量，`response_done` 表示一次模型响应完成。它们是 SDK Model 边界的结构，不要求与上游 HTTP 流中的事件名完全相同。

收到文本增量时，`RecordedModel` 使用稳定条目 ID 累积内容，并写入 `content.delta` 事件。响应完成时，`complete()` 将完整输出条目、请求标识和用量交给 `finishModelStep()`。工具调用意图也属于这些输出条目。

“一个 ModelStep 对应一次模型调用尝试”是产品的建模约定。严格计费时仍需对照传输层记录：步骤在网络发送前持久化，所以发送失败可能留下步骤；客户端的内部重试也可能增加真实请求数。本实现显式关闭所配置的模型自动重试，以减少隐藏调用，但真实请求数量仍以观察证据为准。

## 模型输出一个工具调用，循环为什么还会继续

Tool Calling（工具调用）从模型返回一个结构化意图开始。例如，以下是按 SDK 规范化形状说明的合成调用：

```json
{
  "type": "function_call",
  "callId": "call-example-1",
  "name": "search_documents",
  "arguments": "{\"content\":\"查询等待期条款\",\"query\":\"示例保障甲 等待期\",\"companyIds\":null,\"productIds\":null,\"topK\":5}"
}
```

`callId` 把调用意图与结果连接起来。模型在这一步结束了当前响应，但整个用户任务仍然需要工具返回内容。SDK 找到名为 `search_documents` 的函数工具并调用它。

Harness 的工具包装器随后检查调用身份、版本和权限，创建 ToolInvocation（工具执行实例），执行后端，并把结果持久化。工具完成后，SDK 将结果加入下一次模型输入，再次请求模型。[^tools]

```text
ModelStep S1：模型提出查询意图
  S1 completed
    → ToolInvocation V1 started
    → ToolResult 保存
    → V1 succeeded
ModelStep S2：模型读取工具结果并生成答案
  S2 completed
```

这说明 ModelStep 完成与任务完成是不同层次。一次模型响应可以没有工具、包含一个工具，或包含多个工具。工具的具体执行状态由自己的实体承载，而不是把模型步骤的结束时间延长到所有外部动作结束。

SDK 负责这段工具循环；产品门面负责让循环与持久状态、授权和预算保持一致。[^sdk]

## 最终输出还需要一次产品提交

`runner.run()` 返回流式运行对象。工程既消费事件迭代器，也等待 `stream.completed`，并在错误处理上及时建立监听，避免完成 Promise 的失败被遗漏。随后检查运行仍未被取消，并从已持久化的助手条目中取得最终引用。[^facade]

```ts
// 省略重复取消检查和心跳收尾后的源码摘编。
await completed;
controller.signal.throwIfAborted();

const finalContentRef = facts.finalAssistantIds.at(-1);
if (!finalContentRef) {
  throw new RunBoundaryError('FINAL_CONTENT_UNAVAILABLE');
}

await this.storage.finishRun(current.lease, {
  status: 'completed',
  finalContentRef,
  usage: { elapsedMs: Math.floor(performance.now() - started) },
});
```

`finalContentRef` 指向已经保存的最终助手内容。`finishRun()` 负责把这个引用与当前 Turn、Run 终态和事件共同提交。

这个最后的动作不能仅凭“模型返回了文本”就跳过。如果取消命令在模型结束前已经持久成立，存储层会在同一状态锁下处理相应条件。若 Worker 丧失租约，也不能继续以旧资格覆盖监督逻辑的结果。

## 用一条受控轨迹验证连接处

要验证整个循环，适合使用 Scripted Model（按脚本返回响应的模型替身）。它第一次返回固定工具调用，第二次检查收到的工具结果并返回固定答案。工具使用合成数据，但队列、存储、SDK 和控制面保持真实实现。

这样的测试至少需要观察：受理得到一个 Run；Worker 成功领取；第一个 ModelStep 提供调用条目；ToolInvocation 保存结果；第二次模型请求能读取结果；最终引用指向已经提交的助手条目。

进一步可以在工具结果写入时注入错误。如果下一次模型请求仍然发生，就说明持久化边界没有真正约束循环。这种测试比读取一段漂亮的对话输出更接近 Harness 的工程责任。

项目的 Runner 实现说明记录了真实 PostgreSQL、pg-boss 和 SDK 受控模型的纵向测试，也包含结果提交失败、持久取消和并发工具预算的验证。它们证明的是执行机制，真实模型自主选择哪条业务路线需要另一组行为实验。[^tests]

## 执行系统的核心是一组有顺序的边界

从 Job 到最终答复，运行先经过任务领取，再经过模型调用边界、工具执行边界和终态提交边界。每个边界都明确输入、写入资格和完成条件。

这让后台运行可以与 HTTP 请求寿命分离，也让模型循环真正成为产品执行的一部分。下一次模型该看到什么，则由 Session 和上下文投影决定：[会话历史与模型上下文](05-session-and-context.md)。

## 实现依据

[^worker]: Insurance Agent Runtime 源码快照 `c66112dfd8676b47b143dc2e26739278cf4b0dde`：`apps/runtime/src/runner/worker.ts`，`RunWorker.start` 与监督循环。
[^lease]: 同一快照：`apps/runtime/src/storage/store.ts`，`claimRun`、`assertLease`、`withLease`、`heartbeat`。数据库有效期使用 `clock_timestamp()`。
[^facade]: 同一快照：`apps/runtime/src/runner/facade.ts`，`RunnerFacade.execute`。示例代码均为对应函数摘编。
[^model]: 同一快照：`apps/runtime/src/runner/model.ts`，`RecordedModel.begin`、`complete`、`getStreamedResponse`。
[^tools]: 同一快照：`apps/runtime/src/runner/tools.ts`，`createRunTools` 与 `customDataExtractor`。
[^tests]: `docs/implementation/runner-preparation.md` 记录的受控纵向验证；本文未将教学轨迹标为新取得的真实模型运行。
[^sdk]: [OpenAI Agents SDK：Running agents](https://openai.github.io/openai-agents-js/guides/running-agents/)。工程实例固定使用自身锁定版本，具体 SDK 参数与持久化位置以源码为依据。
