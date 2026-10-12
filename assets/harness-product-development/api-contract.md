# 附录 A：通用 Harness API 契约参考

> 配套《基于 Harness 开发完整 Agent 产品》 · 文档版本 1.0 · 2026-10-12
>
> [返回指南](guide.md) · [开发与验收](roadmap.md) · [技术依据](references.md)

## 1. 契约的性质与采用方式

本附录给出可以进一步转为接口定义、运行时校验和契约测试的参考语义。方法、HTTP 路径、字段和默认值是本指南的约定，采用者应在项目中正式确认。本文不是外部标准，也不是已经实现的服务器。

接口分三层：**方法语义**说明业务操作；**传输绑定**说明请求如何到达；**版本与能力**说明部署实际支持到什么范围。文中的 JSON 是完整的示例对象，TypeScript 是结构与接口示例，业务校验和执行由实现方补齐。

### 1.1 能力集合

| 集合 | 能力 | 声明条件 |
| --- | --- | --- |
| `core` | 创建会话、发送、查询、等待、快照、订阅、结果 | 所有主要路径具备契约测试 |
| `control` | 停止、审批、继续 | 实际执行边界及恢复能力经过验证 |
| `branching` | 历史分支、编辑运行 | 上下文边界及原子提交经过验证 |
| `workspace` | 文件、工具、产物 | 环境、权限与外部效果范围明确 |
| `presentation` | 派生视图及可选 UIMessage | 版本、快照和增量规则明确 |
| `administration` | 后台管理、保留及迁移 | 单独的管理权限 |

产品可以逐阶段开放能力；不支持的方法返回明确能力错误。发布“完整参考配置”时，应列出已经验证的集合、平台和例外。

## 2. 连接、版本和可信上下文

`GET /v1/health` 只提供必要的存活及就绪信息。需要身份、工作区或模型配置的详情通过已认证的能力接口获取。

`GET /v1/capabilities` 示例：

```json
{
  "apiVersion": "1.0",
  "eventSchemaVersion": "1.0",
  "profiles": ["core", "control", "branching", "workspace"],
  "control": {
    "stop": true,
    "resume": {
      "supported": true,
      "requiresCompatibleCheckpoint": true,
      "createsNewAttemptAfterSettlement": true
    }
  },
  "history": {
    "editBoundary": "before_user_input",
    "workspaceRollback": false
  },
  "delivery": {
    "formats": ["canonical-batch-v1", "view-batch-v1"],
    "resumeCursor": true,
    "maxHoldMs": 25000
  }
}
```

这些值是示例，不代表任何 SDK 默认提供所有能力。身份过期、资源授权变化和协议不兼容都应中止相关读取或命令，并返回明确原因。

AppContext（可信应用上下文）由服务端建立，包含访问主体、授权范围、工作区、执行环境及策略版本。请求只能选择自己有权访问的资源，不能自行声明管理员角色、可信审批或更高预算。

## 3. 身份、版本与命令幂等

### 3.1 标识用途

| 字段 | 用途 | 注意事项 |
| --- | --- | --- |
| `threadId` | 会话身份 | 与操作系统线程不同 |
| `branchId` | 上下文分支 | 查询及执行都明确选择 |
| `turnId` | 任务身份 | 可包含多个执行尝试 |
| `attemptId` | 本次实际执行身份 | 控制和等待绑定此值 |
| `stepId` | 模型调用身份 | 不等同于 SDK 的整段运行 |
| `intentId` | 模型工具意图 | 保留提供方原始关联 |
| `invocationId` | 工具执行实例 | 与请求、状态、结果绑定 |
| `operationId` | 需要去重的业务副作用 | 安全重试时保持语义稳定 |
| `commandId` | 一次业务命令身份 | 客户端在首次请求前生成 |
| `eventId` | 持久事件身份 | 重发时保持稳定 |
| `traceId` | 诊断追踪身份 | 不承担业务唯一性 |

`branchRevision` 保护分支前沿和输入结构；`controlRevision` 保护控制及生命周期转换；`contentRevision` 或片段偏移管理流式内容。不要让每个文本片段都使用户的控制版本失效。

### 3.2 幂等作用域

本参考绑定使用 `(authenticatedPrincipalScope, commandId)` 作为命令唯一作用域。操作名称、目标资源和业务请求内容共同参与语义摘要。因此，同一主体范围内不能把相同命令标识复用于另一个操作；`getCommand` 可以通过主体和标识唯一找回回执。客户端等待模式、单次网络超时及诊断标识不进入业务摘要。

同作用域、同内容返回原回执；同作用域、不同内容返回 `IDEMPOTENCY_CONFLICT`。身份变化后先重新授权，不能仅凭知道命令标识读取原结果。

后台在事务内处理并发重复请求，由唯一约束保护。幂等记录保留期必须公开；过期后长期离线客户端应先查任务，不能假定旧标识永久防止重复。涉及长期副作用时可保存较长的操作标记或禁止模糊重放。

## 4. 公共方法与 HTTP 绑定

| 方法 | 参考路径 | 含义 |
| --- | --- | --- |
| `getCapabilities` | `GET /v1/capabilities` | 版本及实际能力 |
| `registerWorkspace` | `POST /v1/workspaces` | 建立工作区与环境关系 |
| `createThread` | `POST /v1/threads` | 创建会话及主分支 |
| `listThreads` | `GET /v1/threads` | 受权范围内分页查询 |
| `getThread` | `GET /v1/threads/{threadId}` | 会话元数据和分支 |
| `send` 的受理步骤 | `POST /v1/threads/{threadId}/turns` | 持久化任务并返回回执 |
| `getCommand` | `GET /v1/commands/{commandId}` | 查询丢失响应的回执 |
| `getTurn` | `GET /v1/turns/{turnId}` | 当前任务和尝试状态 |
| `getAttempt` | `GET /v1/turns/{turnId}/attempts/{attemptId}` | 指定执行结果 |
| `wait` 的单次等待 | `GET /v1/turns/{turnId}/attempts/{attemptId}/wait` | 有界等待一个条件 |
| `stop` | `POST /v1/turns/{turnId}/stop` | 停止指定尝试 |
| `resume` | `POST /v1/turns/{turnId}/resume` | 从合法基准继续 |
| `decideApproval` | `POST /v1/approvals/{approvalId}/decisions` | 处理审批 |
| `forkBranch` | `POST /v1/threads/{threadId}/branches` | 建立历史分支 |
| `editAndRun` | `POST /v1/threads/{threadId}/edits` | 原子建立分支和新任务 |
| `getSnapshot` | `GET /v1/threads/{threadId}/snapshot` | 指定分支/格式快照 |
| `subscribe` | `GET /v1/threads/{threadId}/events` | 可恢复的有序批次 |
| `getArtifact` | `GET /v1/artifacts/{artifactId}` | 产物元数据及受控访问 |

查询分页使用不透明游标并绑定排序及过滤条件。后台管理、删除数据和工作区回滚分别使用管理接口或独立能力，不能混入普通 `stop`。

## 5. 任务受理与输入契约

### 5.1 请求

```json
{
  "commandId": "cmd_100",
  "branchId": "branch_main",
  "expectedBranchRevision": 12,
  "input": {
    "text": "检查当前项目的类型错误并修复",
    "attachments": [
      {"artifactId": "artifact_spec", "version": "1"}
    ]
  },
  "execution": {
    "agentConfigRef": "agent_default_v3",
    "budgetProfile": "interactive_standard"
  }
}
```

服务端校验会话和分支关系、附件归属、配置与预算选择权限、分支前沿及活动任务。首个参考配置在同分支活动任务冲突时返回 `BRANCH_BUSY`，不会隐式打断或替换任务。

支持排队的配置需要明确排队输入基于哪个上下文、何时确定前沿，以及用户是否可修改或取消队列。

### 5.2 回执

```json
{
  "commandId": "cmd_100",
  "threadId": "thread_1",
  "branchId": "branch_main",
  "turnId": "turn_21",
  "attemptId": "attempt_1",
  "acceptedAt": "2026-10-12T08:00:00.000Z",
  "branchRevision": 13,
  "status": "queued",
  "cursor": "cur_opaque_100"
}
```

受理成功意味着输入、任务、初始尝试、待执行项和回执已可靠提交。HTTP 绑定返回 202 表达已受理；它不声称业务工作已经完成。[S01](references.md#s01)

`attemptId` 在受理事务中分配，即使任务尚未开始也存在。这样，排队期间的停止和等待仍有明确目标。

### 5.3 同步响应模式

公共客户端的 `send(input, {responseMode: "async"})` 返回回执；`sync` 模式在同一回执上继续调用 `wait`，返回回执及等待结果。

本参考 HTTP 绑定始终把任务提交与结果等待分开。其他传输可在一次调用中保持等待，但必须具有相同幂等及查询语义。调用方在响应丢失时以原 `commandId` 找回任务，而非生成新命令再次执行。

## 6. 等待契约：保持目标和时间语义稳定

### 6.1 输入

```json
{
  "turnId": "turn_21",
  "attemptId": "attempt_1",
  "until": "settled",
  "waitTimeoutMs": null
}
```

`settled` 表示指定尝试产生不可变的最终处理记录。`settled_or_attention` 另外允许在等待审批或需要人工恢复时返回关注状态。

默认 `waitTimeoutMs: null` 表示逻辑等待不设置短期限；仍受调用方取消、授权失效和永久连接错误影响。`0` 表示立即读取，正数表示整个逻辑等待允许的时长。

等待只能观察，不能延长任务预算或推进新的工具执行。关闭观察连接不会隐式调用停止接口。

### 6.2 有界传输与逻辑等待

单次 HTTP 等待使用 `maxWaitMs`，服务端可按连接策略缩短本段等待，返回 `pending` 和续接信息。客户端库持续等待同一 `attemptId`，按原逻辑期限计算剩余时间。

参考逻辑：

```text
持有任务回执与目标尝试
  → 查询已提交的尝试结果
  → 结果存在：返回 settled
  → 达到用户指定逻辑期限：返回 timeout
  → 否则执行有界等待
  → 单段到期或可重试断线：退避后继续观察同一目标
```

实现应通过“检查状态—建立监听—再次检查”或持久化游标读取避免丢失唤醒。多个等待者共享观察能力，不创建多个执行者。调用方取消等待时，返回带回执的 `WAIT_ABORTED` 或对应客户端异常。

### 6.3 返回值

正常结束示例：

```json
{
  "kind": "settled",
  "turnId": "turn_21",
  "attemptId": "attempt_1",
  "outcome": "completed",
  "resultRef": "attempt_result_1",
  "taskOutcome": "succeeded",
  "finalContentRef": {"stepId": "step_54", "itemKey": "content_1"},
  "artifactIds": ["artifact_patch"],
  "settledAt": "2026-10-12T08:02:10.000Z"
}
```

显式等待超时示例：

```json
{
  "kind": "timeout",
  "turnId": "turn_21",
  "attemptId": "attempt_1",
  "status": "waiting_approval",
  "approvalIds": ["approval_9"],
  "cursor": "cur_opaque_145"
}
```

`outcome` 取 `completed`、`failed`、`stopped` 或 `recovery_required`。`taskOutcome` 表示用户目标的结果，可取 `succeeded`、`partial`、`unsatisfied`、`unknown`。

`recovery_required` 需要返回未确认的工具和资源范围。此结果表示本次自动执行已交由恢复流程处理，不表示远端副作用已被撤销。

等待审批默认不是 `settled`。非交互 CLI 需要返回给脚本时，应显式使用 `settled_or_attention`，而不是把审批暂停伪装成完成。

## 7. 停止与恢复契约

### 7.1 停止请求

```json
{
  "commandId": "cmd_stop_1",
  "attemptId": "attempt_1",
  "reason": "user_requested"
}
```

服务端确认这是当前可控制尝试，记录停止意图，返回命令回执和当前状态。已正常完成时返回已存在的结果；目标已被新尝试替代时返回 `TARGET_ATTEMPT_CHANGED`。

停止回执不是清理完成证据。需要等待停止结果时，继续等待原 `attemptId`。原始用户任务可以保留部分内容、工具结果和产物。

### 7.2 恢复请求

```json
{
  "commandId": "cmd_resume_1",
  "fromAttemptId": "attempt_1",
  "expectedControlRevision": 7,
  "expectedBranchRevision": 13
}
```

恢复前检查兼容检查点、分支前沿、资源占用、未知副作用和当前权限。成功时保留 `turnId`，返回新的 `attemptId`，并把原尝试作为历史保留。

审批等待后的正常继续通常仍属同一尝试，由审批处理器驱动。SDK 的一次恢复调用可以是同一尝试中的执行片段，也可以对应新的产品尝试；映射由产品生命周期决定。

### 7.3 不可恢复的情况

| 条件 | 结果 |
| --- | --- |
| 存在无法确认的工具副作用 | `RECOVERY_REQUIRED`，先进入核对 |
| 检查点版本或配置不兼容 | `CHECKPOINT_INCOMPATIBLE` |
| 原分支已有冲突后续历史 | `RESUME_CONFLICT`，给出分支选择 |
| 当前资源权限失效 | `FORBIDDEN` 或资源隐藏策略对应错误 |
| 工作区内容已变化 | `WORKSPACE_CHANGED`，提供差异及处理要求 |
| Task 已明确完成 | `TURN_ALREADY_COMPLETED`，重做使用新任务 |

从未知效果恢复到可继续状态，应产生独立的核对记录和明确决策，不改写原尝试曾经出现未知结果的历史。

## 8. 审批接口与条件绑定

客户端先读取经过授权和脱敏的审批详情：工具、请求摘要、资源范围、执行环境、风险说明、有效期及版本。

提交示例：

```json
{
  "commandId": "cmd_approve_1",
  "expectedApprovalRevision": 1,
  "requestDigest": "sha256:example-request-digest",
  "decision": "approve",
  "scope": "once"
}
```

服务端验证当前主体的审批资格、目标是否仍然等待、摘要及版本是否匹配。批准和拒绝竞争时，只有一个有效决定；重复相同命令返回原结果。

实际执行前重新校验资源、策略和请求。如果脚本文件、工作区或重要环境变化影响已批准的操作，原决定需要重新评估。摘要只是绑定机制的一部分，不能替代对可变资源的检查。

已序列化的执行状态只保存在可信后台。客户端提交审批标识与决定，不上传一个由自己修改的“已批准执行快照”。SDK 状态的反序列化并不构成身份或审批真实性校验。[S06](references.md#s06)

## 9. 历史分支、编辑和回退

### 9.1 分支位置

参考配置采用 `before_user_input` 边界：选定某条用户输入，分支继承它之前的稳定上下文。历史工具调用和结果需要保持协议完整。

分支查询返回稳定 `branchId`、父分支、分叉位置、版本和可用操作。用户界面中的“1/2”等序号由这些关系派生。

### 9.2 原子编辑运行

```json
{
  "commandId": "cmd_edit_1",
  "sourceBranchId": "branch_main",
  "targetTurnId": "turn_12",
  "expectedSourceRevision": 20,
  "input": {
    "text": "只修复类型错误，保留当前功能行为",
    "attachments": []
  }
}
```

`editAndRun` 在一个事务内建立新分支、保存新输入、创建新任务与尝试、更新回执。目标旧输入不再进入新分支，因此不会把原输入和修改后输入重复提供给模型。

原历史保留。源分支存在活动执行时，参考配置返回 `BRANCH_BUSY`；需要先停止并确认资源状态，或使用具有隔离工作区的扩展能力。

### 9.3 回退预览

回退预览只改变当前客户端的历史选择和草稿。取消预览不改变服务端执行事实。确认发送后使用分支或编辑接口。

工作区回滚有独立的预览、权限、版本前置条件和结果记录；本参考基础契约只承诺对话分支。能力发现应清楚表达是否实现文件回滚。

## 10. 快照与规范化事件

### 10.1 快照输入与输出

请求明确 `branchId`、`format`、协议版本和必要过滤条件。快照示例：

```json
{
  "threadId": "thread_1",
  "branchId": "branch_main",
  "branchRevision": 13,
  "format": "view-batch-v1",
  "projectionVersion": "conversation-layout-1",
  "cursor": "cur_opaque_145",
  "activeAttempt": {
    "turnId": "turn_21",
    "attemptId": "attempt_1",
    "status": "waiting_approval"
  },
  "items": [],
  "approvalIds": ["approval_9"]
}
```

`items` 在实际响应中包含调用方有权读取的视图项，空数组只为简化示例。快照的游标必须对应实际包含的位置，而不是后台最新但尚未投影的事件。

### 10.2 权威事件外壳

```json
{
  "eventId": "event_146",
  "schemaVersion": "1.0",
  "threadId": "thread_1",
  "branchId": "branch_main",
  "turnId": "turn_21",
  "attemptId": "attempt_1",
  "sequence": "146",
  "type": "tool.completed",
  "recordedAt": "2026-10-12T08:01:00.000Z",
  "causationId": "event_145",
  "data": {
    "intentId": "intent_7",
    "invocationId": "invocation_7",
    "resultRef": "result_7"
  }
}
```

`sequence` 是指定会话事实流内的提交顺序，以十进制字符串传输以避免大整数精度问题。并发事件的序号说明存储顺序，因果关系另由标识体现。

提供方时间作为来源元数据保留。`recordedAt` 由可信记录服务产生。按权限过滤后，调用方收到的事件不必数值连续。

### 10.3 事件分类

| 分类 | 示例 | 语义 |
| --- | --- | --- |
| 命令与任务 | `turn.accepted`、`attempt.started` | 受理及执行生命周期 |
| 模型调用 | `model.step.started`、`model.step.completed` | 本次调用边界 |
| 内容 | `content.appended`、`content.committed` | 按身份和偏移保存内容 |
| 工具 | `tool.requested`、`tool.started`、`tool.completed` | 意图与实际执行 |
| 审批 | `approval.requested`、`approval.decided` | 请求和授权事实 |
| 控制 | `stop.requested`、`attempt.settled` | 控制意图与最终处理记录 |
| 历史 | `branch.created` | 已提交版本关系 |
| 恢复 | `recovery.reconciled` | 外部状态核对和决定 |

事件名称是本指南的参考定义。未知事件根据其重要性采取忽略附加元数据、升级客户端或强制重建；影响状态的未知事件不能静默跳过后继续声称一致。

## 11. 批次、游标与重连协议

### 11.1 基础 SSE 形式

```text
id: cur_opaque_146
event: harness.batch
data: {"batchId":"batch_146","afterCursor":"cur_opaque_145","nextCursor":"cur_opaque_146","events":[]}

```

实际 `events` 包含一个有界且可完整应用的批次。数据量超过限制时，内容存放于受控引用或拆成有完整性边界的小批次。半条 SSE 消息或未完成批次在重连后重新读取。

SSE 定义传输事件和 `Last-Event-ID` 等机制，应用需要自己保证历史重放和已应用状态的一致性。[S03](references.md#s03)

### 11.2 客户端应用规则

验证访问范围、格式版本及 `afterCursor`；对批次身份去重；完整应用该批次后再保存 `nextCursor`。客户端可以把状态与游标放在同一持久化提交中，或崩溃后重新从服务器快照重建。

客户端不能将网络已读位置直接当作业务已应用位置。原生事件源的自动重连行为与应用事务之间，需要适配器明确协调。

### 11.3 服务端补齐规则

服务端按有效游标读持久记录，补齐后继续读取新增提交。订阅唤醒丢失时应有再检查机制。游标绑定主体、分支、过滤条件、格式和版本；错误范围返回 `CURSOR_SCOPE_MISMATCH`，历史过期返回 `RESYNC_REQUIRED`。

敏感权限撤销后停止发送，必要时要求客户端清除或重新获取缓存。旧游标本身不授予继续读取的权利。

### 11.4 可选 UIMessage 输出

UIMessage 的文本、工具、自定义数据等片段可由展示适配器产生。[S09](references.md#s09) 产品需要另外定义任务状态与游标的提交标记，声明协议版本，并验证从历史中途加入及片段重复的行为。

`[DONE]` 只结束当前传输，消息 `finish` 只结束相应消息。客户端仍以产品的 `attempt.settled` 或状态查询判断任务处理结果。

最简可靠实现可先使用 `view-batch-v1` 同步完整派生状态，再把视图交给 UIMessage 组件；无需让组件库负责权威同步和执行控制。

## 12. 工具结果、产物与内容范围

工具结果区分执行状态、业务返回状态、标准输出、诊断输出、退出码、结果引用和可能的外部效果。退出码为零不单独证明用户任务已经完成。

产物接口至少返回 `artifactId`、版本、内容类型、大小、校验信息、来源任务、访问条件及保留状态。下载时再次授权；签名地址按有效期使用并避免进入普通日志。

输出分页或范围读取需要稳定内容版本。尚在增长的工具日志显式返回当前长度和终结标记；内容被截断、脱敏或过期时提供状态，避免把无权读取误显示为空结果。

## 13. 错误模型

HTTP 错误可采用 `application/problem+json`，按 RFC 9457 表达类型、状态和说明，并增加稳定业务代码。[S02](references.md#s02)

```json
{
  "type": "urn:harness:problem:branch-conflict",
  "title": "分支版本已变化",
  "status": 409,
  "code": "BRANCH_REVISION_CONFLICT",
  "detail": "请读取当前分支后重新决定操作。",
  "commandId": "cmd_edit_1",
  "currentRevision": 21,
  "retryable": false,
  "traceId": "trace_501"
}
```

`retryable` 表示相同语义请求能否安全重试；状态冲突往往需要重新读取并决定，不能机械重发。

| 错误 | 处理方向 |
| --- | --- |
| `VALIDATION_ERROR` | 修正输入 |
| `UNAUTHENTICATED` | 重新认证 |
| `FORBIDDEN` / `NOT_FOUND` | 按资源隐藏策略处理访问问题 |
| `UNSUPPORTED_CAPABILITY` | 选择支持的配置或禁用入口 |
| `IDEMPOTENCY_CONFLICT` | 检查命令标识复用 |
| `BRANCH_BUSY` | 等待、显式停止或选择独立分支 |
| `TARGET_ATTEMPT_CHANGED` | 重新确认要控制的执行 |
| `CHECKPOINT_INCOMPATIBLE` | 升级兼容层或从安全输入新建任务 |
| `RESYNC_REQUIRED` | 重新加载快照 |
| `BUDGET_EXCEEDED` | 根据授权预算和任务结果处理 |
| `RECOVERY_REQUIRED` | 核对外部状态，保留已有证据 |

错误中仅返回当前主体有权访问的对象与版本。栈、密钥、原始请求和内部存储地址不默认返回客户端。

## 14. 三种版本与兼容规则

**API 版本**管理公共请求与结果。**事件版本**管理事实或展示批次结构。**投影版本**管理相同事实怎样组织成视图。数据库及 SDK 检查点另有内部版本。

新增可选字段通常可以兼容；修改状态含义、默认等待终点、身份作用域、取消效果和分支边界，需要显式版本决策及迁移测试。

客户端启动时协商能力；更新或重连后再次检查。旧客户端不能因为能解析 JSON 就认为语义兼容。

## 15. 核心结构示例

以下 TypeScript 片段定义公共语义，生产实现还需完整的字段结构、运行时校验、网络适配和错误处理。

```ts
export type AttemptStatus =
  | "queued" | "running" | "waiting_approval" | "stopping"
  | "completed" | "failed" | "stopped" | "recovery_required";

export type AttemptOutcome =
  "completed" | "failed" | "stopped" | "recovery_required";

export interface ExecutionTarget {
  turnId: string;
  attemptId: string;
}

export interface CommandReceipt extends ExecutionTarget {
  commandId: string;
  threadId: string;
  branchId: string;
  acceptedAt: string;
  branchRevision: number;
  cursor: string;
}

export type WaitResult =
  | (ExecutionTarget & {
      kind: "settled";
      outcome: AttemptOutcome;
      settledAt: string;
      resultRef: string;
    })
  | (ExecutionTarget & {
      kind: "timeout" | "attention_required";
      status: AttemptStatus;
      cursor: string;
    });

export interface WaitOptions {
  until?: "settled" | "settled_or_attention";
  waitTimeoutMs?: number | null;
  signal?: AbortSignal;
}

export interface HarnessObserver {
  wait(target: ExecutionTarget, options?: WaitOptions): Promise<WaitResult>;
}
```

`resultRef` 指向完整处理结果；前述 JSON 例子同时展开了摘要字段。`signal` 取消本地等待，实际停止通过独立方法提交。

接口代码可以生成客户端类型，但类型系统不会认证调用方、保证事务、阻止重复副作用或验证授权。对应行为由服务端及执行环境实现。

## 16. 契约检查清单

采用者应能够回答：受理在哪个提交点成立；重复请求如何返回；同步等待绑定哪个尝试；审批何时算完成；停止清理不确定时返回什么；继续如何检查分支和工作区；快照和事件怎样衔接；游标与权限怎样绑定；历史编辑怎样保证原子性；产物如何受控读取。

这些答案形成机器可验证的结构、状态测试和端到端用例。具体测试编号及阶段归属见[开发路线与验收矩阵](roadmap.md)。
