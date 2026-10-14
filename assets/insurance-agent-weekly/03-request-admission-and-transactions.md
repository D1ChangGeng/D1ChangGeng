# 用户点击发送之后：请求受理、幂等与执行事务

顾问查询一份条款，页面迟迟没有返回，于是又点了一次发送。服务器实际可能已经受理了第一次请求，只是回执在网络上丢失了。此时，如果第二次请求再次创建任务，系统会运行两次模型、读取两次业务服务，还可能在同一段对话里产生两份答案。

这个问题发生在模型调用之前。它属于 Control Plane（控制面）：一组负责创建、查询和控制运行对象的接口。一个可靠的 Harness（智能体执行控制系统）首先要决定，什么叫“已经接受了这次工作”，以及调用方重试请求时应该得到什么。

本文沿 Insurance Agent Runtime 的 HTTP 入口和 `RuntimeStorage.submitTurn()` 展开。为了观察真实行为，我们把条款内容换成合成输入，只关注请求身份、数据库状态和队列任务。

## 把“发送消息”定义为一个命令

在这套产品里，Thread（对话线程）表示持续会话，Turn（用户回合）表示一次受理的用户输入，Run（执行尝试）表示完成该回合的一次运行。调用方先创建 Thread，再向它提交 Turn。

```http
POST /v1/threads/{threadId}/turns
Authorization: Bearer <受控测试令牌>
Content-Type: application/json
Idempotency-Key: query-demo-request-001

{
  "input": { "text": "请查询示例保障甲的等待期" },
  "agentProfileId": "synthetic-profile",
  "expectedThreadRevision": 0
}
```

`synthetic-profile` 是控制面测试中的配置名称；实际实例应使用服务端开放的已授权 Profile。请求形状来自工程的 `SubmitTurn` 契约。成功受理后，返回值包含以下字段：[^control]

```json
{
  "threadId": "示意：Thread 标识",
  "turnId": "示意：Turn 标识",
  "runId": "示意：Run 标识",
  "threadRevision": 1,
  "eventsPath": "/v1/threads/示意线程/events"
}
```

这些标识在实际运行中由服务端生成。`202` 表示输入、执行对象和任务已经持久受理；模型结果通过 Run 状态、内容读取和事件接口随后提供。

请求中的三个业务字段各有职责：`input` 保存问题，`agentProfileId` 选择服务端允许的运行配置，`expectedThreadRevision` 表示调用方基于哪个线程版本提交。模型凭证、上游地址和机构身份则由服务端建立。

## 入口校验既检查形状，也检查身份

`createControlHttpApp()` 建立 Fastify 应用，挂载请求解析、认证和路由。Fastify 的 Schema（结构约束）负责请求形状，产品认证器提供 Principal（已认证主体），其中包含 `orgCode`、`subjectId` 和 `scopes`。[^http]

这三个字段分别确定机构、用户或服务主体，以及授权操作范围。查询某个 Thread 时，存储层再次检查该资源是否属于当前主体。这样，即使调用方构造了一个存在的 Thread ID，也不能越过归属边界。

工程还处理了一个看似细小的入口问题：敏感请求头重复出现时，哪一个值有效？不同代理和库对重复头的合并可能不同，因此实现直接拒绝重复的 `authorization`、`idempotency-key`、`if-match` 和 `last-event-id`。这让命令身份只有一种解释。

另一个细节是框架自动转换。实现关闭 `removeAdditional`、`coerceTypes` 和 `useDefaults`，再用权威解析器处理输入。于是，“请求字段原本非法”可以得到明确反馈，不会先被框架悄悄删除或改成另一种类型。[^http]

## 幂等键把网络重试连接到原命令

Idempotency（幂等）在这里表示：相同命令被重复提交时，返回同一个受理结果。它的关键不是“输入文字相同”，而是“调用方明确声明这是同一个命令”。

工程使用以下作用域构成幂等身份：

```text
机构 + 主体 + 操作范围 + Idempotency-Key
```

其中操作范围还包含对应 Thread 等资源标识。请求正文经过规范化后计算摘要，供同键重放时比较。

如果找到了原幂等记录，服务端先检查当前资源仍可被访问，再比较正文摘要。同键、同正文返回原回执；同键、不同正文返回 `IDEMPOTENCY_CONFLICT`。如果没有原记录，才执行新的受理动作，并把回执与正文摘要一起保存。[^storage]

并发条件下，“先查记录，再创建”还不够。两个事务可能同时查到不存在。实际 `idempotent()` 使用事务级 PostgreSQL advisory lock（应用定义的事务锁），先串行化同一作用域的处理，再查询幂等记录。

```ts
// 源码摘编：真实代码还包括归属复核与回执保存。
const scope = [
  principal.orgCode,
  principal.subjectId,
  operation,
  key,
];

await tx.execute(sql`
  SELECT pg_advisory_xact_lock(
    hashtextextended(${JSON.stringify(scope)}, 0)
  )
`);
```

锁由数据库事务释放。它保护的既是已有记录的重放，也是“记录尚未出现”的并发窗口。数据库中保存的幂等回执，则让这种保证跨越进程重启。

## 一次受理需要共同提交哪些内容

`submitTurn()` 的核心工作不是立即调用模型，而是建立一份可以由后台安全领取的持久任务。实现把以下内容放在同一个事务中：

```text
运行配置快照
用户输入条目
Turn
queued 状态的 Run
turn.created 和 run.queued 事件
pg-boss 队列 Job
Thread revision 更新
幂等请求回执
```

这组内容必须一致。例如，只有 Run 没有 Job，任务可能永久停在排队；只有 Job 没有 Run，Worker 无法知道它的授权范围和输入；事件先于事务结果对外可见，则外部系统可能观察到一个最终回滚的任务。

工程通过 `fromDrizzle(tx, sql)` 将 pg-boss 的任务发送接到当前事务。它复用同一 PostgreSQL 事务边界，而不是先提交业务数据库再单独调用另一个队列。[^storage]

```ts
// 源码摘编：Job 与 Run 使用同一事务提交。
const job = await this.boss.send(
  this.queueName,
  { runId },
  { id: queueJobId, db: fromDrizzle(tx, sql) },
);

if (job !== queueJobId) {
  throw new StorageError('INVALID_STATE');
}
```

`queueJobId` 与 `runId` 是两个身份。前者属于队列任务，后者属于产品执行。Worker 收到的 Job 数据中携带 `runId`，由此领取产品运行对象。

这解决了当前单数据库部署中的双写窗口。将来若使用独立消息系统，需要重新设计可靠投递协议，不能把“我们现在放在同一事务里”的保证直接迁移过去。

## 同一个 Thread 只允许一个活动 Run

假设顾问在前一条查询尚未完成时又发送一条追问。系统需要明确这是排队、插入当前上下文，还是拒绝并请调用方等待。该实现采用每个 Thread 同时一个活动 Run 的规则，活动状态覆盖排队、运行以及等待取消确认。[^tests]

受理时取得 Thread 行锁，检查忙状态和版本。若线程已经有活动执行，返回 `THREAD_BUSY`；其他 Thread 仍可以各自受理请求。

这是一种有意识的首版取舍。它让会话历史的可见边界保持清楚：一个回合完成后，下一个回合从已经接受的历史继续。运行中插入新消息属于另一套调度语义，需要单独定义输入顺序、上下文截取和取消行为。

`expectedThreadRevision` 则解决另一个问题：调用方是否基于过期线程状态提交。忙状态反映“正在执行什么”，revision（资源版本）反映“你看到的状态是否仍有效”。两者具有不同意义，错误码也分别表达。

## 一次真实修复：先重放回执，再解析新的配置

假设第一次请求已经受理，随后模型配置暂时不可用。调用方为了拿回丢失的回执，重放原请求。如果入口每次都先解析模型和工具配置，这次重放会因为当前配置故障而失败，尽管原 Run 早已存在。

因此，工程把 `resolveSnapshot()`（解析运行快照）放在幂等重放、资源权限和线程状态检查之后。只有一个真正的新命令才解析可信配置。旧命令重放直接返回当时保存的身份。[^tests]

测试中有一个很直接的做法：第一次请求让解析器成功，随后把解析器改为抛错，再重复同一个请求。预期是第二次依然返回相同 `202` 回执，而且解析器调用次数保持一次。对于同键不同正文和线程忙的请求，也检查解析器没有再次执行。

这个案例说明，校验的先后顺序本身就是产品语义。把检查项全部列出来，还不足以保证行为正确；必须把它们放在合适的事务和重放边界上。

## 让测试观察数据库，而不只检查状态码

一个接口返回 `202`，只能证明响应满足了某种形状。要验证真正的受理行为，测试还应沿回执读取产品状态和队列。

工程中的集成测试使用真实 PostgreSQL 和 pg-boss，并断言重复 Turn 请求只产生一个 Run、一个用户条目，以及能按 Job ID 读回的真实队列任务。以下片段保留了测试的关键判断：

```ts
// 摘编自 tests/control/http.test.ts；fixture 是项目测试夹具。
const first = await submit(thread, 'same-turn-key');
expect(first.statusCode).toBe(202);

const replay = await submit(thread, 'same-turn-key');
expect(replay.json()).toEqual(first.json());

expect(await scalar(
  'SELECT count(*) AS n FROM runtime.runs',
)).toBe(1);

expect(await scalar(
  "SELECT count(*) AS n FROM runtime.runtime_items WHERE role='user'",
)).toBe(1);
```

并发测试同时提交两个不同命令，预期一个 `202`、一个 `409 THREAD_BUSY`。跨机构和跨主体测试还覆盖历史、导出、取消和 Fork（从历史创建新线程），确保资源归属规则贯穿所有入口。[^tests]

测试环境也很重要。数据库用例应连接专属测试资源，创建隔离临时数据库。仅运行跳过数据库组的测试集合，不能证明这条事务链已经实际执行。

## 从一个 HTTP 回执得到可靠的执行起点

请求受理完成后，系统已经拥有用户输入、权限范围、冻结配置、队列任务和可追溯身份。模型尚未开始回答，但工作已经成为一个明确的产品对象。

这一层建立的保证很朴素：一次网络重试仍然指向原工作，一个失败事务留下统一结果，一个 Thread 的历史按既定规则推进。后续 Worker、模型循环和事件输出，都以这个起点为依据。

下一步是观察后台如何取得执行资格，以及租约为何要伴随每次状态写入：[Worker 与模型循环](04-worker-and-model-loop.md)。

## 实现依据

[^control]: Insurance Agent Runtime 源码快照 `c66112dfd8676b47b143dc2e26739278cf4b0dde`：`apps/runtime/src/storage/store.ts` 的 `submitTurn`，以及 `tests/control/http.test.ts` 中的请求和响应。示意 ID 不代表真实运行记录。
[^http]: 同一快照：`apps/runtime/src/transport/http/app.ts` 的 `createControlHttpApp`、`rejectDuplicateCommandHeaders`；请求认证、Schema 与错误映射由该入口连接。
[^storage]: 同一快照：`RuntimeStorage.idempotent`、`prepareSnapshot`、`submitTurn`、`readyThread`。数据库锁的语义可参阅 [PostgreSQL：Explicit Locking](https://www.postgresql.org/docs/16/explicit-locking.html)。
[^tests]: 同一快照：`tests/control/http.test.ts` 中关于 same-key creation、repeated Turn requests、unavailable snapshots、one writer per Thread 和 org/subject isolation 的测试；以及 `docs/implementation/storage-preparation.md`。本文分析已有源码与测试，未将示意请求写作一次新执行的结果。
