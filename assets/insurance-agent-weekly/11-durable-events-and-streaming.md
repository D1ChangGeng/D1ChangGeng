# Harness 最终输出什么：持久事件、内容引用与 SSE

一段模型文本正在持续返回。外部调用方已经看见“查询到相关条款”，连接却突然断开。重新连接时，它需要知道后面发生了什么：工具是否完成，模型是否继续调用，任务最终成功还是取消，以及已经看到的文字会不会重复出现。

这些问题要求一种比“把模型输出直接转发出去”更稳定的协议。Harness（智能体执行控制系统）需要先定义自己的运行事实，再把事实转换为可查询、可续传的外部事件。

本文沿 Insurance Agent Runtime 的 `runtime_events` 和 `DurableEventStreams` 实现，解释从数据库状态到 SSE（Server-Sent Events，服务器发送事件）的完整路径。讨论止于服务端输出与协议消费语义，不涉及界面布局。

## 模型事件与产品事件属于不同层

模型适配器可能返回文本增量、工具调用条目和响应结束等事件。它们描述一次模型调用的进展。产品还需要描述请求受理、Run（执行尝试）启动、工具结果提交、用户取消和检查点创建。

因此，模型事件经过产品边界处理后，才形成 Runtime Event（运行事件）。例如：

```text
模型文本增量
  → 更新当前助手内容条目
  → 保存 content.delta

工具结果提交
  → 保存结果条目与 ToolInvocation 终态
  → 保存 tool_invocation.completed

运行完成
  → 保存 Turn.finalContentRef 与 Run 终态
  → 保存 turn.completed 和 run.completed
```

在这套实现中，事件跟随对应数据库事务提交。事件的意义来自已经成立的产品变化，而不是来自某个 SDK 回调的名字。[^storage]

## 事件账本并不等于完整事件溯源架构

Event Ledger（事件账本）在这里指按顺序持久保存的运行变化。产品同时有 Thread、Turn、Run、ModelStep、ToolInvocation 和运行条目表，实体状态与事件在事务中共同维护。

因此，不能只因为存在一张事件表，就假设所有数据库状态都可以仅靠事件从零重建。Event Sourcing（事件溯源）要求事件具有足够完整的权威语义，并把状态作为事件派生结果；本文分析的工程采用的是事务状态与持久事件共同工作的设计。

对外事件可以携带引用，让调用方按需查询详细对象。工具完成事件例如只需要指出 invocation ID 和 result item ID，而不必把所有私有上下文塞进每一帧。

这减少了内容重复，也把公开输出和内部 SDK 状态区分开。

## 一个事件需要哪些稳定字段

实际 `RuntimeEvent` 使用共同信封和按类型约束的 payload（事件负载）。下面是一条合成示例：[^contracts]

```json
{
  "schemaVersion": "1.0",
  "eventId": "SYN-EVENT-18",
  "threadId": "SYN-THREAD-1",
  "seq": "18",
  "turnId": "SYN-TURN-1",
  "runId": "SYN-RUN-1",
  "occurredAt": "2026-01-01T00:00:00Z",
  "type": "tool_invocation.completed",
  "payload": {
    "invocationId": "SYN-INVOCATION-1",
    "resultItemRef": "SYN-ITEM-RESULT-1"
  }
}
```

`eventId` 标识这一事件，`seq` 表示它在 Thread（对话线程）中的顺序。两者不需要使用同一种值。`runId` 和 `turnId` 则让调用方把事件定位到执行尝试与用户回合。

时间戳用于描述发生时间，事件排序以 `seq` 为依据。并发动作可能具有非常接近的时间戳，甚至在不同系统中出现时钟误差；明确的线程内序号更适合续传和去重。

序号在 PostgreSQL 中使用 bigint（大整数），在 JSON 中使用十进制字符串。JavaScript 的普通 number 无法精确表达所有 64 位整数，因此把大序号转换成 number 再比较，会破坏协议身份。

```js
// 独立运行的精度示例。
const after = 9007199254740993n;
const next = 9007199254740994n;
console.log((next > after), next.toString());
```

服务端和消费方都应保留字符串或使用合适的大整数类型，而不是依靠显示格式修正精度丢失。

## 内容条目与最终引用怎样保持一致

模型输出文本时，`RecordedModel` 按输出条目标识累计内容，并保存 `content.delta`。增量包含 `contentRef`、`delta` 和 `offset`。`contentRef` 将每次增量连接到同一个产品内容条目。[^model]

此快照中的 offset 使用 JavaScript 字符串长度计算，也就是 UTF-16 代码单元数；它不是 UTF-8 字节数。跨语言消费时应明确这个实现约定。例如：

```js
console.log('😀'.length);                   // 2 个 UTF-16 代码单元
console.log(Buffer.byteLength('😀', 'utf8')); // 4 个 UTF-8 字节
```

输入输出的字节预算可以继续按 UTF-8 计算，文本偏移则要遵守协议的实际位置单位。混用单位会让重放后的文字错位。

模型响应完成后，完整输出条目进入 committed（已提交）状态。Run 真正结束时，`finalContentRef` 指向已经提交的助手条目。`finishRun()` 核对引用属于当前回合和合法执行历史，再提交 Turn 与 Run 终态。[^storage]

源码同时支持传入新 `finalItem` 或已有 `finalContentRef`。前一路径会显式产生 `content.committed` 事件，已有引用路径则复用已保存内容。因此，观察某条具体运行时，不应强行要求所有完成路径都出现完全相同的内容事件序列。权威判断应结合条目状态、最终引用和运行终态。

## SSE 是持久事件的传输方式

SSE 使用 `text/event-stream`，按文本帧持续发送事件。标准字段包括 `id`、`event` 和 `data`。该工程将 Thread 序号作为 SSE 的 `id`，把事件类型写到 `event`，完整事件 JSON 写到 `data`。[^sse]

```text
id: 18
event: tool_invocation.completed
data: {"schemaVersion":"1.0","eventId":"SYN-EVENT-18","threadId":"SYN-THREAD-1","seq":"18","turnId":"SYN-TURN-1","runId":"SYN-RUN-1","occurredAt":"2026-01-01T00:00:00Z","type":"tool_invocation.completed","payload":{"invocationId":"SYN-INVOCATION-1","resultItemRef":"SYN-ITEM-RESULT-1"}}

```

帧之间的空行表示事件边界。心跳采用以冒号开头的 comment（注释帧），保持连接活跃，但不分配产品事件序号。

`DurableEventStreams.open()` 在写入 HTTP 200 流式响应头之前，先读取第一批事件。这一步执行当前授权、资源归属和游标检查。于是认证失败、资源不可见或游标过期，仍然可以返回正常的结构化 HTTP 错误，而不是在已经开始的 SSE 流里临时发明一套错误格式。

## 用同一种查询完成历史重放和实时接续

有一种常见设计是先查询历史，再订阅消息代理。若两步之间产生了新事件，系统必须处理那个交接窗口。

当前实现采用更直接的方式：历史和后续事件都读取同一个持久表。发送完一批之后，以最后序号查询下一批；数据暂时为空时等待一小段时间，再继续查询。[^stream]

```ts
// DurableEventStreams 的核心行为摘编。
let cursor = after;
let batch = firstBatch;

while (!signal.aborted) {
  for (const event of batch) {
    await writeFrame(event);
    cursor = event.seq;
  }
  await waitForNextPollWhenNeeded(batch);
  batch = await read(cursor, batchSize);
}
```

每次查询都可以重新检查当前权限、删除状态和事件保留窗口。没有另一条独立消息渠道，也就没有历史查询与订阅之间的转换窗口。

代价是轮询延迟和数据库读取负载。当前默认轮询间隔为 250 毫秒，批量大小为 100。这是一种明确取舍：先获得简单、可验证的持久接续，再根据实际规模决定是否引入唤醒通知等优化。通知可以减少空轮询，持久序号仍然应该保留为恢复依据。

## 写入网络缓冲区不等于对方已经处理

Node.js 的 `response.write()` 返回 false，表示当前输出缓冲区需要背压处理。Backpressure（背压）让生产者等待消费速度，避免不断堆积数据。

实现等待 `drain` 事件后才继续发送，并在当前批次尚未处理完时暂停获取后续批次。连接拥有独立 AbortController（取消控制器），关闭时能够清理等待和计时器。[^stream]

但是，写入成功只说明数据进入了服务端网络发送路径，不证明消费方已经业务处理完成。连接在任意时刻中断，客户端可能已经收到一帧但还没保存自己的游标，也可能没有收到最后一帧。

因此，可靠消费应保存自己确认处理的位置，重连后通过 `(threadId, seq)` 去重。服务端能够按持久游标重放，不能凭一次 write 回调承诺端到端 exactly once（恰好一次处理）。需要严格业务确认的下游，应在自己的事务或确认协议中管理处理进度。

## 游标过期要有明确恢复入口

事件通常有保留策略。调用方携带的 `after` 或 `Last-Event-ID` 已早于保留窗口时，继续返回后面还存在的事件会让它误以为历史完整。

工程返回 `410 EVENT_CURSOR_EXPIRED`，包含快照位置和当前序号，调用方可以重新读取当前资源状态，再从合适位置继续观察。两个游标来源同时提供时，接口要求它们一致。[^retention]

保留窗口还会与查询竞争。若先读取窗口元数据，再在另一个数据库视图中读取事件，清理任务可能恰好在中间删除一批记录。实现使用只读 repeatable-read（可重复读）事务，使同一次批量查询中的窗口和事件来自一致快照。

这个保证属于当前批次。下一次查询可能观察到新的清理结果，并按最新窗口返回过期状态。对长连接而言，权限和保留范围始终是随时间变化的条件。

## 观察连接和运行本身分别结束

SSE 连接是观察者。关闭连接只释放该订阅的资源，后台 Run 继续遵循自己的预算、队列和租约。用户要结束运行，应提交取消命令，由持久状态驱动执行器收尾。[^stream]

服务关闭时，代码还处理了慢消费方一直不读取的情况：清理订阅并关闭对应流，使等待背压的观察者不会阻止应用停机。这个动作管理的是观察连接；运行器怎样收尾，仍由其独立生命周期控制。

这一区分同样适用于命令行、服务间调用和其他消费方式。观察谁，并不自动拥有对那个运行的取消权。

## 把输出协议作为端到端测试的一部分

事件测试应使用真实数据库和真实流读取，检查每个 SSE 序号与数据库记录一致。中途断开观察连接，确认 Run 仍在执行，再从已处理游标重连，核对剩余事件。

还应覆盖大整数序号、当前授权变化、过期游标、慢读取、并发订阅和最终引用一致性。工程的 `tests/events/` 与实现记录保存了这些测试方向，包括保留窗口竞争和背压关闭。[^retention]

这让事件层拥有独立于界面的验收标准：只要能消费 HTTP/SSE，就可以证明 Harness 对外表达的状态与内部事实一致。

## 一个稳定出口连接整个运行系统

模型生成内容，工具获得证据，存储提交事实，事件记录变化，SSE 负责传输。每一层都保留自己的完成语义，外部系统才有可能在断线、重连和恢复后得到一致认识。

到这里，Harness 已经从可信请求入口走到了稳定事件出口。下一步是验证这些边界在正常和异常路径下都成立：[协议测试与效果评测](12-testing-and-evaluation.md)。

## 实现依据

[^storage]: Insurance Agent Runtime 源码快照 `c66112dfd8676b47b143dc2e26739278cf4b0dde`：`apps/runtime/src/storage/store.ts` 的 `finishModelStep`、`finishInvocation`、`finishRun`、事件写入和读取方法。
[^contracts]: 同一快照：`packages/contracts/src/events.ts`，事件信封及各类 payload Schema。正文样例使用合成标识。
[^model]: 同一快照：`apps/runtime/src/runner/model.ts` 的 `getStreamedResponse` 和 `complete`。offset 单位依据该版本的字符串长度计算方式说明。
[^stream]: 同一快照：`apps/runtime/src/transport/http/sse.ts`，`DurableEventStreams.open`、输出循环、背压和关闭处理。
[^retention]: `docs/implementation/events-sse-preparation.md` 与 `tests/events/` 的既有验证；事务背景参阅 [PostgreSQL：Transaction Isolation](https://www.postgresql.org/docs/16/transaction-iso.html)。
[^sse]: [HTML 标准：Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)。标准说明传输帧与事件 ID；产品游标、权限和持久化语义来自本工程。
