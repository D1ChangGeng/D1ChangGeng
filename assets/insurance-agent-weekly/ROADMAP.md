# Insurance Agent｜14 周 Harness 工程实践分享路线

> 规划状态：课程周次与业务任务按原始 14 周大纲保留；Harness 技术主线按照已开发的 D 方向工程重新组织。  
> Week 01 起始；Week 02 起共有 13 场正式实践分享。实际分享日期按课程日历登记。  
> 本系列工程边界：HTTP / SSE 事件对外输出；产品前端 UI 实现作为独立下游工作。  
> 课程资料及实际保险业务原文遵循授权范围；公开归档仅放置适合发布的脱敏摘要、源码相对路径和可复核测试信息。

## 总任务与双主线

**最终目标：** 构建并说明一个完整单 Agent Harness，支持保险顾问提出业务问题后，从 API 接入、认证和状态受理开始，经过持久队列、Worker、Agents SDK 模型循环、Tool Registry、业务适配、工具结果提交、会话上下文与恢复机制，直到权威事件生成与 SSE 输出。

- **课程业务切入：** Week 01 固定 RAG；Week 02 Gold/基线；Week 03 固定链路诊断；Week 04 最小循环；Week 05 充分性；Week 06 预算；Week 07 对比；Week 08 多跳；Week 09 路由；Week 10 集成；Week 11 可观测；Week 12 调优；Week 13 报告；Week 14 答辩。
- **Harness 主体：** 每周固定选择一段已实现的 Harness 代码深入解释输入、函数调用、数据结构、状态变化、事件和验证。
- **系统出口：** Canonical facts → durable runtime_events → HTTP/SSE。说明输出的数据契约与传输行为，重点保持在 Harness 服务端。
- **周期关系：** 此处 14 周是课程实践分享路线；D 方向 28 自然日工程窗口是已实施工程的开发节奏。各周以真实代码版本和历史验收结果复盘，并持续补充获得授权的新资料与实验。

```text
Client request
  → Fastify / Auth / Thread–Turn–Run Control Plane
  → PostgreSQL transaction (canonical input + snapshot + queued Run + event + pg-boss Job)
  → Worker claim / lease / fence
  → RunnerFacade / RunEnvironment / PgSdkSession
  → RecordedModel (one actual provider request = one ModelStep)
  → SDK chooses function_call
  → Tool Registry / ToolInvocation / Adapter
  → persistent ToolResult / Evidence / sourceRef
  → SDK continues with updated context
  → finalContentRef / run terminal / event ledger
  → HTTP / SSE durable events
```

## 课程周次与 Harness 核心模块总表

| Week | 课程切入 | Harness 深入专题 | M 模块 | 分享 |
| --- | --- | --- | --- | --- |
| 01 | 固定 RAG 基线：环境、现有检索接口、单次问答与 20 问题观察 | **工程入口、组件装配与请求生命周期全景** | M00 / M01 / M05 / M16 | 课程首周；为 Week 02 准备可复核材料 |
| 02 | 60 条人工 Gold、共享约 240 条与固定检索评分基线 | **产品 API 契约、规范化运行条目和测试证据边界** | M02 / M03 / M15 | 正式分享 #1；首次分享的 Harness 主线 |
| 03 | 分析固定 RAG 的一次请求路径和单次检索后的业务不足 | **HTTP Ingress、身份授权、幂等、Thread/Turn/Run 与事务受理** | M03 / M04 | 正式分享 #2 |
| 04 | 最小 Agent 循环，让模型主动调用一次保险检索工具 | **持久队列消费、执行租约、SDK Runner 与 ModelStep** | M01 / M05 | 正式分享 #3 |
| 05 | 充分性自评、证据缺口、补充查询与 TopK 调整 | **Canonical History、PG SDK Session、模型上下文预算与继续执行输入** | M05 / M13 / M14 | 正式分享 #4 |
| 06 | 步骤预算、降级与课程中期交流 | **Run Policy、执行中断传播、持久取消、Retry/Resume 和恢复粒度** | M07 / M14 | 正式分享 #5；中期交流 15 分钟 |
| 07 | 查询分解（一）：两款保险产品的多维度对比 | **ToolCall 关联、多工具调度、结果校验与持久提交** | M05 / M08 / M13 | 正式分享 #6 |
| 08 | 查询分解（二）：跨保司枚举、多文档关联与多跳查询 | **多步骤来源身份、证据持久化、归属校验和上下文关联** | M09 / M10 / M11 / M13 | 正式分享 #7 |
| 09 | 自主工具路由、简单任务快速路径、结构化查询与检索选择 | **ToolSpec / ToolProfile / BackendBindings / ToolsetSnapshot 全链** | M08 / M12 / M13 | 正式分享 #8 |
| 10 | A/B/C/D 集成周：替换检索配置，接入事实/优惠工具与来源映射 | **Composition Root、服务依赖注入、Resource Grant 和业务后端准入** | M09 / M10 / M11 / M12 / M17 | 正式分享 #9；课程集成周 |
| 11 | 可观测：记录每一步查了什么、为何补查、如何结束 | **权威事件、内容增量、顺序游标、回放与对外输出** | M03 / M06 / M14 | 正式分享 #10；本系列的 Harness 对外技术边界 |
| 12 | 端到端调优、固定基线、消融与数据分层分析 | **Scripted Model、Mock Tool、控制事件、Golden Trace 与四层验收** | M15 / M18 | 正式分享 #11 |
| 13 | Agent 设计报告、架构与可复现技术交付 | **工程生命周期、数据库迁移、部署配置、真实支持矩阵与独立审查** | M19 / M20 | 正式分享 #12 |
| 14 | 课程结项答辩：系统演示、可复现实验和成果报告 | **入口到事件的全链验收：Control → Storage → Worker → Model → Tool → Ledger → Events** | M00–M20 | 正式分享 #13；结项答辩 15 分钟 |

## 详细周次定义

### Week 01｜从固定 RAG 到可执行的 Harness：系统边界和最小纵向链路

**课程切入：** 固定 RAG 基线：环境、现有检索接口、单次问答与 20 问题观察。  
**Harness 主体：** 工程入口、组件装配与请求生命周期全景。  
**工程模块：** M00 / M01 / M05 / M16。  
**安排：** 课程首周；为 Week 02 准备可复核材料。

**A. 课程业务问题与原定学习内容**

- 使用课程授权的 rag-open-service 检索接口形成 Question → Retrieval → Evidence → Prompt → Model → Answer 固定链路。
- 运行 20 道真实题型并记录 Query、检索证据、引用、模型输出、耗时及人工观察；按课程授权范围保留原始数据。
- 用固定链路中的一次输入解释：当系统升级为 Agent 时，需要怎样承载长期任务身份与多轮工具动作。

**B. Harness 主体——实际需要讲透的架构、函数与执行逻辑**

- 从 `composeRuntime()` 识别 Fastify API、PostgreSQL、pg-boss、SDK Runner、Registry 与持久事件的装配关系；解释产品 Runtime 与开发协作工具各自的职责。
- 沿请求入口绘制 Thread / Turn / Run → 队列 Job → Worker → SDK → 工具结果 → 终态事件的第一张全链图。
- 明确 SDK 管模型/工具循环、Harness 管认证、执行授权、持久状态、预算、取消与恢复的责任边界。
- 认识合成、录制与真实数据模式，为后续每周复用同一个纵向测试用例创造稳定条件。
- 建立源码 commit、模型身份、工具版本、运行配置与测试结果的工程证据索引。

**C. 本周现场讲解与证据**

- **演示：** 展示一次固定 RAG 问答结果，同时用工程测试追踪一条合成 Run 的状态转移，标注每个关键组件的入口。
- **验收：** 可执行的固定 RAG 程序、20 问题基线记录、端到端组件图、源码/命令身份与最小纵向运行证据。
- **源码与技术依据：** `docs/development-pack/appendices/week1-course-deliverable.md`；`apps/runtime/src/composition/runtime.ts`；`README.md`；`INSTALL-REPORT.md`

**D. 对最终 Harness 的点题：** 建立整个 Harness 的执行地图；后续每周都在这张地图上深入一个具体边界。

---

### Week 02｜从 PDF Gold 到执行事实：Harness 的契约、实体与可验证性

**课程切入：** 60 条人工 Gold、共享约 240 条与固定检索评分基线。  
**Harness 主体：** 产品 API 契约、规范化运行条目和测试证据边界。  
**工程模块：** M02 / M03 / M15。  
**安排：** 正式分享 #1；首次分享的 Harness 主线。

**A. 课程业务问题与原定学习内容**

- 解释 Gold 中 question、answerability、evidenceRequirements、原始 PDF 身份、物理页码与证据类型。
- 展示正文、表格与多文档标注差异，计算 AnyHit@K、MRR@K、EvidenceRecall@K、RequirementCoverage；完成状态以实际人工记录为准。
- 确定固定查询基线的输入、结果与数据集版本，为后续自主 Agent 对照保留公共标尺。

**B. Harness 主体——实际需要讲透的架构、函数与执行逻辑**

- 解释为何建立独立的 Thread、Turn、Run、ModelStep、ToolInvocation、runtime_items 与 event 身份，形成确定性的观察粒度。
- 分析 OpenAPI / TypeScript / Zod / JSON Schema 的生成链与实际路由一致性检查，说明 25 项操作如何构成产品边界。
- 区分模型或工具返回的原始条目、产品保存的 Canonical Item（权威条目）和对外公开的业务结果投影。
- 追踪一份评测输入或运行导出如何绑定源码、运行模式、模型/工具配置、数据集与原始证据引用。
- 将可重复的 Harness 行为测试与基于人工 Gold 的业务质量评测分开记录，建立 T0–T3 四层验证标准。

**C. 本周现场讲解与证据**

- **演示：** 拿同一道保险条款问题，展示 PDF 人工证据、检索返回、标准化评分单元及对应 Harness 执行记录字段。
- **验收：** 一条 Gold 标注样本与 PDF 页核对；一次评分器执行；一张 Thread/Turn/Run/Step/Invocation 实体关系图；契约 drift 检查。
- **源码与技术依据：** `docs/development-pack/appendices/week2-golden-set.md`；`docs/development-pack/docs/03-thread-control-plane.md`；`packages/contracts/generated/openapi.yaml`；`docs/implementation/evaluation-preparation.md`

**D. 对最终 Harness 的点题：** 将整个系统的观察对象和验收对象固定下来，使后续 Harness 行为能够量化比较与复核。

---

### Week 03｜用户点击发送之后：Control Plane 如何创建并接纳一次 Agent 执行

**课程切入：** 分析固定 RAG 的一次请求路径和单次检索后的业务不足。  
**Harness 主体：** HTTP Ingress、身份授权、幂等、Thread/Turn/Run 与事务受理。  
**工程模块：** M03 / M04。  
**安排：** 正式分享 #2。

**A. 课程业务问题与原定学习内容**

- 选取一次固定检索难以覆盖的保险问题，重放传统请求 → 检索 → 回答的具体步骤。
- 提出 Agent 所需的持续任务身份、运行状态和补充查询能力，确定本次业务动作的最小外部 API。

**B. Harness 主体——实际需要讲透的架构、函数与执行逻辑**

- 从 Fastify `createControlHttpApp()` 追踪请求的路由匹配、schema 校验、Principal（机构/主体/权限范围）解析。
- 深入 `POST /v1/threads` 与 `POST /v1/threads/{threadId}/turns`：输入、`agentProfileId`、`expectedThreadRevision`、Idempotency-Key。
- 逐行分析 `RuntimeStorage.submitTurn()` 的执行顺序：权限、幂等/请求摘要、Thread 行锁、revision、RunSnapshot 解析。
- 解释单事务提交 user item、Turn、queued Run、事件、队列 Job 和幂等回执，及失败时一致回滚。
- 解释 `THREAD_BUSY`、`IDEMPOTENCY_CONFLICT`、跨机构资源不可见与 ETag / revision 在高并发请求中的作用。
- 检查受理成功与最终执行完成是两个不同生命周期时刻，建立后续 Worker 追踪入口。

**C. 本周现场讲解与证据**

- **演示：** 使用真实 HTTP 测试展示首发 `202`，重复同键同请求返回相同实体、同键异请求 `409`、同 Thread 竞争只有一个活动 Run。
- **验收：** 一份 API 请求/响应与数据库行关联表；至少三种控制面断言；与同事务 pg-boss Job 对照。
- **源码与技术依据：** `apps/runtime/src/transport/http/app.ts`；`apps/runtime/src/storage/store.ts`；`docs/implementation/control-http-preparation.md`；`tests/control/http.test.ts`

**D. 对最终 Harness 的点题：** 最终任务的第一个强边界：用户请求转化为可信、持久、可调度的 Run。

---

### Week 04｜Agent 真正开始工作：Worker、RunnerFacade 与模型/工具循环

**课程切入：** 最小 Agent 循环，让模型主动调用一次保险检索工具。  
**Harness 主体：** 持久队列消费、执行租约、SDK Runner 与 ModelStep。  
**工程模块：** M01 / M05。  
**安排：** 正式分享 #3。

**A. 课程业务问题与原定学习内容**

- 把 `knowledge.search_documents` 接成 SDK 可调用函数工具，使模型对一条保险条款问题选择查询参数。
- 对比固定 RAG 的单次调用和模型选择工具后继续推理的实际消息序列。

**B. Harness 主体——实际需要讲透的架构、函数与执行逻辑**

- 由 pg-boss Job 的 Run ID 进入 `RunWorker.start()`，观察 `claimRun()`、owner、租约 fence、心跳与过期扫描。
- 剖析 `RunnerFacade.execute()` 如何装配本次 `RunEnvironment`、私有依赖、模型实例、SDK Session、工具集合与 AbortSignal。
- 追踪官方 `Agent`、`Runner.run()` 的运行方式；区分 SDK 内部 turn 和产品自身的 Turn / ModelStep。
- 理解 `RecordedModel.begin()` 在模型请求前校验预算/当前授权并持久化 ModelStep，`complete()` 在响应到达时提交响应事实。
- 沿 function_call 的 callId 追踪模型意图、工具分派、工具结果回流和后续模型请求；把 `ToolInvocation` 独立于 `ModelStep`。
- 检查 SDK 继续执行前所需的 ToolResult 持久化边界及最终终态提交。

**C. 本周现场讲解与证据**

- **演示：** 现场用一段脱敏真实轨迹或受控 scripted model 回放 `Run queued → claimed → ModelStep1 → tool call → tool result → ModelStep2 → final`。
- **验收：** 一条 Run/ModelStep/ToolInvocation 关联链、队列和租约实际 PG 断言、模型/工具重复身份关联测试。
- **源码与技术依据：** `apps/runtime/src/runner/worker.ts`；`apps/runtime/src/runner/facade.ts`；`apps/runtime/src/runner/model.ts`；`apps/runtime/src/runner/tools.ts`；`tests/runner/`

**D. 对最终 Harness 的点题：** 从持久任务领取到模型自主动作为止，讲清 Harness 的核心执行生命周期。

---

### Week 05｜为什么能够继续查：SDK Session、上下文投影和跨步骤状态延续

**课程切入：** 充分性自评、证据缺口、补充查询与 TopK 调整。  
**Harness 主体：** Canonical History、PG SDK Session、模型上下文预算与继续执行输入。  
**工程模块：** M05 / M13 / M14。  
**安排：** 正式分享 #4。

**A. 课程业务问题与原定学习内容**

- 选一个首次检索缺少条款要件的案例，让 Agent 识别具体缺口，并解释查询重写和 TopK 从 5 到 10 的动机。
- 对比首次与后续检索结果的证据新增量、重复查询和可回答范围。

**B. Harness 主体——实际需要讲透的架构、函数与执行逻辑**

- 拆解 `PgSdkSession` 的 `getItems`、`addItems`、`popItem`、`clearSession`，说明 SDK 历史协议与 canonical runtime_items 的引用关系。
- 研究用户输入、上一轮已完成历史、当前工具 call/result 怎样形成下一次 ModelRequest 的输入。
- 通过 `projectModelRequest()` 与 `context-budget.ts` 解释模型可见上下文裁剪与完整审计事实的职责区分。
- 说明 ToolResult 中的 gaps、sourceRefs 和 partial 等字段如何成为下一次模型选择工具的证据。
- 解释继续还是结束主要由模型在当前证据下决定，Harness 提供调用空间、边界与终态规则。
- 使用同一 Thread 的第二个 Turn，检验历史读取、重复文本身份和最终内容引用保持一致。

**C. 本周现场讲解与证据**

- **演示：** 对比模型第一次请求与第二次请求的输入摘要，指出新增 ToolResult、历史引用、预算裁剪和自主补充查询。
- **验收：** 同 Thread 连续问答、工具结果回流、Session pop/clear、重复内容不同身份、上下文上限验证。
- **源码与技术依据：** `apps/runtime/src/runner/session.ts`；`apps/runtime/src/runner/context-budget.ts`；`apps/runtime/src/runner/model.ts`；`docs/implementation/context-budget-preparation.md`

**D. 对最终 Harness 的点题：** 完整任务中模型多次调用的连续性：运行事实稳定保存，模型输入按当次预算和证据动态构造。

---

### Week 06｜Agent 能控制到哪里：预算、取消、超时与检查点恢复

**课程切入：** 步骤预算、降级与课程中期交流。  
**Harness 主体：** Run Policy、执行中断传播、持久取消、Retry/Resume 和恢复粒度。  
**工程模块：** M07 / M14。  
**安排：** 正式分享 #5；中期交流 15 分钟。

**A. 课程业务问题与原定学习内容**

- 对相同保险条款任务分别设置充足与紧凑预算，观察补查次数、证据完整度和可解释结束。
- 说明证据不足时如何明确返回已核验范围和剩余缺口。

**B. Harness 主体——实际需要讲透的架构、函数与执行逻辑**

- 细分 SDK Turn 上限、模型请求上限、工具调用预算、工具并发、上下文字节、证据字节、Run 总时间和结果字节。
- 追踪 `RunnerFacade` 创建 AbortController、设置 deadline、心跳读取持久取消意图以及将 signal 传递给模型和工具。
- 解释 cancellation request（持久取消请求）与 cancellation acknowledgement（执行取消确认）的先后关系。
- 比较 Retry（同一 Turn 的新 Run）与 Resume（从已批准 Checkpoint 创建新 Run）的数据关联。
- 分析已支持的 `tool_result_boundary`、`sdk_state`、`turn_boundary` 三种恢复边界及 SDK/toolset/model/version 的兼容性。
- 通过租约 fence 与过期 Run 扫描复盘 worker 退出、tool outcome_unknown 与已提交工具结果复用。

**C. 本周现场讲解与证据**

- **演示：** 演示预算耗尽或模型/工具执行中取消，读取持久 Run 状态、部分条目、检查点及后续可恢复范围。
- **验收：** 运行时限、模型/工具预算、取消竞争、旧租约拒写、worker 重启与三类恢复边界断言。
- **源码与技术依据：** `apps/runtime/src/runner/facade.ts`；`apps/runtime/src/runner/recovery.ts`；`apps/runtime/src/storage/recovery.ts`；`docs/implementation/recovery-boundaries-fix.md`；`tests/recovery/`

**D. 对最终 Harness 的点题：** 完成自主执行的治理面：让每个 Run 在预算、错误与中断条件下仍具有可审计终态。

---

### Week 07｜复杂对比是怎样执行的：ToolInvocation 生命周期与多工具协作

**课程切入：** 查询分解（一）：两款保险产品的多维度对比。  
**Harness 主体：** ToolCall 关联、多工具调度、结果校验与持久提交。  
**工程模块：** M05 / M08 / M13。  
**安排：** 正式分享 #6。

**A. 课程业务问题与原定学习内容**

- 将两产品对比拆为产品身份、赔付定义、最低保费、适用限制等证据需求。
- 让模型选择目录、检索、事实或来源查询，并形成逐项证据关联的比较结果。

**B. Harness 主体——实际需要讲透的架构、函数与执行逻辑**

- 拆解模型 output 中多个 function_call 的 callId 与 `ModelStep` 关联方式，解释为何多个调用可以属于同一次模型响应。
- 追踪 `createRunTools()` 中的工具名唯一性、执行 effect、版本校验、requiredScopes、输入参数和 content 分离。
- 解释工具执行前预算预留、`startInvocation()`、调用执行、Schema / 结果字节验证、sourceRefs 汇集。
- 研究 `customDataExtractor` 中先持久化结果条目和 ToolInvocation 成功状态、再允许 SDK 继续模型循环的顺序。
- 通过 `maxFunctionToolConcurrency` 和共享工具预算观察多工具并发与结果顺序，保留每项调用独立状态。
- 分析失败、取消、超时、权限拒绝与 `outcome_unknown` 怎样形成规范化工具结果事实。

**C. 本周现场讲解与证据**

- **演示：** 同一模型步骤返回两个工具意图：逐个查看 invocationId、callId、工具结果和后续继续执行，插入一次持久化失败反例。
- **验收：** 单/多工具调用数、并发峰值、权限校验、结果 schema、commit-before-continue 和不确定结果状态的测试。
- **源码与技术依据：** `apps/runtime/src/runner/tools.ts`；`apps/runtime/src/runner/model.ts`；`apps/runtime/src/runner/types.ts`；`tests/runner/`；`docs/implementation/runner-preparation.md`

**D. 对最终 Harness 的点题：** 从模型行动意图到可追踪 ToolInvocation 的治理机制，为后续复杂业务执行复用。

---

### Week 08｜多跳证据如何保持可信：Canonical Items、SourceRef 与授权来源

**课程切入：** 查询分解（二）：跨保司枚举、多文档关联与多跳查询。  
**Harness 主体：** 多步骤来源身份、证据持久化、归属校验和上下文关联。  
**工程模块：** M09 / M10 / M11 / M13。  
**安排：** 正式分享 #7。

**A. 课程业务问题与原定学习内容**

- 使用保司目录和产品目录进行覆盖范围确定，再通过检索与原文页读取补齐各对象证据。
- 明确哪些问题需要完整枚举、哪些需要由第一轮输出决定下一轮查询，保留未完成对象集合。
- 展示一组原始 PDF 与解析页的物理页码映射，以及跨文件引用一致性。

**B. Harness 主体——实际需要讲透的架构、函数与执行逻辑**

- 沿 `catalog.list_insurers` → `catalog.list_products` → `knowledge.search_documents` → `knowledge.get_source_page` 重建多跳的真实 ToolInvocation 图。
- 解释 `runtime_items` 的 canonical 内容身份、ModelStep/Invocation 引用和 Session/history 复用关系。
- 拆解 Evidence 中 rank、score、chunkId、documentId、orgDocId、page、metadataStatus 的独立语义。
- 理解持久 `sourceRef` 如何绑定机构/用户、文档版本、允许页和来源授权，而读取时再次确认权限。
- 演示部分元数据、源文档未准入、产品 ID 映射变化和解析表格警告如何作为缺口传回工具结果。
- 验证页面来源引用与 ToolResult、最终答复之间的关联可以从持久事实回溯。

**C. 本周现场讲解与证据**

- **演示：** 以脱敏跨产品问题构造 ModelStep → ToolInvocation → SourceRef → 原文页的关联链，展示已核对的页码与权限结果。
- **验收：** 多文档范围覆盖、目录分页、来源跨机构隔离、元数据缺口、已授权原文页读回与引用一致性。
- **源码与技术依据：** `docs/implementation/source-content-cache.md`；`docs/implementation/evidence/M09-real-component-gate-20261007.json`；`docs/implementation/evidence/M10-real-component-gate-20261007.json`；`docs/implementation/evidence/M13-original-dod-gate-20261008.json`

**D. 对最终 Harness 的点题：** 保障多跳行动链每一步都有明确业务对象、实际证据、授权来源和持久关联。

---

### Week 09｜如何让模型自己选工具：Tool Registry 的规格、Profile 与运行快照

**课程切入：** 自主工具路由、简单任务快速路径、结构化查询与检索选择。  
**Harness 主体：** ToolSpec / ToolProfile / BackendBindings / ToolsetSnapshot 全链。  
**工程模块：** M08 / M12 / M13。  
**安排：** 正式分享 #8。

**A. 课程业务问题与原定学习内容**

- 给同一 Agent 输入条款解释、精确金额查询、产品枚举三种题型，观察工具选择和查询次数。
- 比较结构化事实查值与向量检索的业务适配性，并记录简单题的调用时延与复杂题的补查价值。

**B. Harness 主体——实际需要讲透的架构、函数与执行逻辑**

- 逐层分析 ToolSpec（名称/版本/输入输出/效果）、ToolProfile（模型可见集合）、BackendBindings（实际实现）与 ToolsetSnapshot（本次 Run 固定能力）。
- 解释 `capabilityId`、模型工具名、版本、Schema、effect、scopes、timeout、handler/binding key 的作用。
- 追踪当前 Principal、Profile、Run snapshot 和当前权限取交集后如何决定真实可执行工具集合。
- 说明能力定义与后端实现分别版本化，快照摘要怎样影响运行一致性及检查点恢复兼容。
- 演示 `content` 作为公开动作描述，业务参数交给受控服务端 binding，机构和凭证由本地环境提供。
- 核对六项只读业务能力（检索/来源/保司/产品/事实/优惠）在 synthetic/recorded/real 的同契约入口。

**C. 本周现场讲解与证据**

- **演示：** 为同一 Run 使用两个不同 profile，比较目录可见性、实际授权和模型能发起的工具；变更工具版本后检查恢复兼容判定。
- **验收：** 工具清单/详情、scope 隔离、Schema、ToolsetSnapshot/hash、bindingVersion 与版本冲突测试。
- **源码与技术依据：** `docs/implementation/registry-runtime-integration.md`；`apps/runtime/src/runner/tools.ts`；`docs/development-pack/docs/05-tool-registry.md`；`tests/registry/registry.test.ts`

**D. 对最终 Harness 的点题：** 把模型动作选择连接到经过授权、具有版本和执行契约的真实业务能力。

---

### Week 10｜四方向集成的工程实质：Composition、Port/Adapter 与 Real/Mock

**课程切入：** A/B/C/D 集成周：替换检索配置，接入事实/优惠工具与来源映射。  
**Harness 主体：** Composition Root、服务依赖注入、Resource Grant 和业务后端准入。  
**工程模块：** M09 / M10 / M11 / M12 / M17。  
**安排：** 正式分享 #9；课程集成周。

**A. 课程业务问题与原定学习内容**

- 梳理 A 的来源与索引字段、B 的结构化事实/优惠契约、C 的检索配置，明确需要 D 接受的版本和数据含义。
- 使用同一个问题分别执行合成/录制/已准入真实能力，观察工具返回与可用证据的差异。

**B. Harness 主体——实际需要讲透的架构、函数与执行逻辑**

- 从 `composeRuntime()` 进入依赖注入：Storage、HTTP、Worker、模型 Provider、工具实现与维护任务各在哪里装配。
- 明确 Port 是领域类型契约、Adapter 是实际网络/SDK实现、Service 保存当前授权、缓存和业务规则。
- 拆解同一 ToolSpec 下的 synthetic/recorded/real BackendBinding 选择及 Resource Grant（资源授权）/readback（实际读回）验证。
- 分析模型/资源/来源在 Profile 与 RunSnapshot 中冻结的配置和版本摘要，结合首次真实调用前的当前授权复核。
- 追踪 Retrieval、Source、Catalog、Facts 与 Promotions 的不同上游协议、空值/partial/UNKNOWN、分页和时效性返回。
- 建立跨方向接口契约测试与真实 readback 的分层门禁，按每项资源状态激活可用能力。

**C. 本周现场讲解与证据**

- **演示：** 通过一条真实或录制的保险业务调用，展示 ToolSpec 保持一致、Binding 发生变化时输入输出、版本状态和权限校验的完整路径。
- **验收：** Port/Adapter 契约、跨机构隔离、当前授权复核、resource digest/readback、Real/Mock parity、跨方向能力支持矩阵。
- **源码与技术依据：** `apps/runtime/src/composition/runtime.ts`；`docs/implementation/real-business-composition-preparation.md`；`docs/implementation/facts-promotions-handoff.md`；`docs/implementation/evidence/delivery-support.current.json`

**D. 对最终 Harness 的点题：** 使业务能力从开发夹具进入受控真实环境，保持 Harness 契约和执行链一致。

---

### Week 11｜Harness 最后输出什么：Canonical Fact、Event Ledger 与持久 SSE

**课程切入：** 可观测：记录每一步查了什么、为何补查、如何结束。  
**Harness 主体：** 权威事件、内容增量、顺序游标、回放与对外输出。  
**工程模块：** M03 / M06 / M14。  
**安排：** 正式分享 #10；本系列的 Harness 对外技术边界。

**A. 课程业务问题与原定学习内容**

- 选取一次多次检索的 Run，列出每次查询、补查说明、来源变化、预算状态和最终答复。
- 用事件记录检查任务执行过程与对外状态是否一致。

**B. Harness 主体——实际需要讲透的架构、函数与执行逻辑**

- 梳理 UserItem、ModelStep、ToolInvocation、runtime_items、finalContentRef 与 runtime_events 的单一事实身份和关联。
- 按事务提交顺序追踪 `run.queued/started`、`model_step.started/completed`、`tool_invocation.*`、`content.delta/committed`、`checkpoint.created` 与终态事件。
- 说明每个 Thread 的 `seq` 单调增长、事件 ID 与 `seq` 区分、十进制字符串保持 PostgreSQL bigint 精度。
- 深入 `DurableEventStreams`：先授权预检，再重放历史事件，然后以当前 seq 接续持久尾流。
- 解释 `after` / `Last-Event-ID`、过期游标 `410`、snapshot 定位、心跳、背压和慢消费方资源释放。
- 验证 SSE 观察连接生命周期独立于后台 Run，最终内容/历史/导出共同引用已提交的 finalContentRef。

**C. 本周现场讲解与证据**

- **演示：** 从数据库事件序列逐条对照 SSE `id/event/data` 帧，断开观察连接后重连并核对遗漏事件与最终内容引用。
- **验收：** 事件顺序/幂等、bigint 精度、断线续传、cursor 过期、并发观察者、背压、取消与终态一致性。
- **源码与技术依据：** `apps/runtime/src/transport/http/sse.ts`；`apps/runtime/src/transport/http/app.ts`；`apps/runtime/src/storage/store.ts`；`docs/implementation/events-sse-preparation.md`；`tests/events/`

**D. 对最终 Harness 的点题：** 完成从内部执行事实到稳定外部事件协议的交付；系列主体技术边界止于 HTTP/SSE 输出。

---

### Week 12｜怎样证明 Harness 真的可靠：协议测试、故障注入与 Agent 效果归因

**课程切入：** 端到端调优、固定基线、消融与数据分层分析。  
**Harness 主体：** Scripted Model、Mock Tool、控制事件、Golden Trace 与四层验收。  
**工程模块：** M15 / M18。  
**安排：** 正式分享 #11。

**A. 课程业务问题与原定学习内容**

- 对比固定一次检索与自主继续查询在相同问题集上的检索覆盖和成本差异。
- 建立问题类型、证据完整性、来源支持、模型生成质量各自的评测口径。

**B. Harness 主体——实际需要讲透的架构、函数与执行逻辑**

- 把不稳定的模型输出替换为 scripted model API response，固定 function_call、返回内容、stop reason 和工具结果。
- 注入 Control Events（控制事件）序列：create/submit、cancel、retry、resume、fork、delete 及并发争用。
- 使用 Golden Trace（预期轨迹）检查已受理命令、ModelStep/Invocation 顺序、canonical 事实、事件和终态不变量。
- 针对竞争、权限撤回、网络超时、写入失败、worker 退出、恢复版本变化进行可重复故障注入。
- 划分 T0 工程机制、T1 真实接口、T2 真实模型动作、T3 人工 Gold 业务效果，分别提交证据。
- 使用 TypeScript 导出和 Python 评分实现可追溯比较，记录版本、预算、用量、重复查询、Evidence Coverage 和质量指标。

**C. 本周现场讲解与证据**

- **演示：** 复现一个 ToolResult 写入失败或取消竞争的真实测试：展示预期执行轨迹、实际 DB/事件事实与测试断言，再展示固定基线/自主系统的指标差异。
- **验收：** 确定性端到端回归、故障注入报告、Golden Trace/invariant 矩阵、真实资源 readback、独立模型与业务效果报告。
- **源码与技术依据：** `docs/development-pack/docs/11-evaluation.md`；`docs/implementation/evaluation-preparation.md`；`docs/implementation/evidence/acceptance-current-standing.json`；`tests/runner/`；`tests/recovery/`；`tests/events/`

**D. 对最终 Harness 的点题：** 以明确的机制正确性和业务质量测量证明 Harness 在真实执行条件下的能力。

---

### Week 13｜一个可交付的 Harness 长什么样：版本、运维、恢复与发布证据

**课程切入：** Agent 设计报告、架构与可复现技术交付。  
**Harness 主体：** 工程生命周期、数据库迁移、部署配置、真实支持矩阵与独立审查。  
**工程模块：** M19 / M20。  
**安排：** 正式分享 #12。

**A. 课程业务问题与原定学习内容**

- 把保险咨询链路的业务工具、查询策略、证据来源和已评测效果整合到正式设计报告。
- 整理 B/C/A 对接版本、授权来源、语义质量状态和后续扩展位置。

**B. Harness 主体——实际需要讲透的架构、函数与执行逻辑**

- 由 API 入口、运行 Worker、Registry、PG Schema、SDK Session、事件协议收敛最终模块图和职责表。
- 解释 25 项控制操作、14 张数据库表、数据库迁移、版本化 Schema 和工具/来源支持矩阵如何对应真实发布。
- 重放 Docker/Node 部署、启动/停止、数据库迁移、备份/新库恢复、留卷重启的独立工程验证。
- 梳理源码 commit/tree、容器镜像 digest、测试命令、观察窗口、ACS source readback 和交付清单之间的证据身份。
- 区分 21 个工程模块门禁、最终源码封存、实际资源准入、模型质量与成本结论各自的验收范围。
- 输出面向后续应用的 Harness API 契约、事件接口、权限与版本边界、故障恢复操作说明；事件对外输出即为产品接口边界。

**C. 本周现场讲解与证据**

- **演示：** 从干净源码/受控镜像启动到创建 Run、完成一条合成或授权任务、读回事件、停止/重启后查询同一状态。
- **验收：** 安装/迁移/备份恢复、全回归、镜像与源码身份、支持矩阵、独立审查和发布候选交付清单。
- **源码与技术依据：** `docs/implementation/architecture-decisions.md`；`docs/implementation/D-REPORT.current.md`；`docs/implementation/evidence/d7-release-image-lifecycle.json`；`docs/implementation/evidence/d7-release-acs-readback.json`

**D. 对最终 Harness 的点题：** 将 Harness 从可执行代码收敛为有版本、有证据、有操作手册的工程交付。

---

### Week 14｜完整 Harness 结项演示：从一个保险问题到一组可核验的运行事件

**课程切入：** 课程结项答辩：系统演示、可复现实验和成果报告。  
**Harness 主体：** 入口到事件的全链验收：Control → Storage → Worker → Model → Tool → Ledger → Events。  
**工程模块：** M00–M20。  
**安排：** 正式分享 #13；结项答辩 15 分钟。

**A. 课程业务问题与原定学习内容**

- 演示一个单产品条款查证、一个证据不足后的自主补查、一个跨产品/跨保司复杂问题。
- 用固定评测集说明证据覆盖、模型工具选择、成本和业务质量的已验证结论。

**B. Harness 主体——实际需要讲透的架构、函数与执行逻辑**

- 以真实输入创建 Thread/Turn/Run，核对认证、幂等、事务、队列和执行租约。
- 现场展开 SDK ModelStep 与 ToolInvocation 关联，展示 Registry 授权、实际 Adapter 调用、结果提交和继续执行。
- 检查 Session 历史、Context 预算、SourceRef 证据、finalContentRef 与 Run 最终状态的一致性。
- 演示受控取消、预算结束或 Checkpoint 恢复场景，展示可靠性测试和已持久化事实的保留。
- 从持久 event ledger 读取完整执行轨迹，并通过 HTTP/SSE 观察顺序、重放与最终终态输出。
- 按 T0/T1/T2/T3 汇报工程、资源、模型和业务验收，把可复现源码、报告和运行证据组成最终交付。

**C. 本周现场讲解与证据**

- **演示：** 用 1 条完整真实/授权链路和 1 条受控故障链路覆盖入口到事件：请求 → Run → Worker → ModelStep → ToolInvocation → canonical item → finalContentRef → SSE。
- **验收：** 端到端可复现演示、核心执行不变量、精确版本与真实资源支持矩阵、人工 Gold/质量结果、提交材料与答辩报告。
- **源码与技术依据：** `docs/development-pack/docs/12-acceptance-and-handoff.md`；`docs/implementation/D-REPORT.current.md`；`docs/implementation/evidence/delivery-support.current.json`；源码 `apps/runtime/src/` 全执行链

**D. 对最终 Harness 的点题：** 最终交付是一个具备可信入口、可靠执行、可用工具、证据历史和稳定事件协议的 Insurance Agent Harness。

## 每周统一讲解与材料标准

单次普通分享建议 12–15 分钟；第 6 周中期交流与第 14 周答辩按课程要求预留 15 分钟。

| 环节 | 参考时长 | 必须讲清楚 |
| --- | --- | --- |
| 业务问题引入 | 2 分钟 | 本周课程场景对应完整保险任务中的哪一段 |
| Harness 关键代码与执行链 | 7–8 分钟 | 入口函数、调用对象、状态、数据库事实和实际执行顺序 |
| 实际轨迹与验证 | 3–4 分钟 | 输入、ToolCall、ToolResult、ModelStep、Run 状态、测试断言及 evidence |
| 总结与下周关联 | 1 分钟 | 本周这段 Harness 对最终任务的作用，后续将连接到哪一个模块 |

各周优先准备以下证据文件；公开仓库只保留脱敏内容与授权范围内材料：

```text
weekly-sharing.md             # 实际展示版本
problem-and-scope.md          # 业务任务与 Harness 对应边界
implementation-notes.md      # 函数/代码/架构判断
trace-redacted.json           # 脱敏模型与工具执行轨迹
validation-summary.md        # 测试断言、结果、源码和环境身份
```

每周内容固定回答：① 业务输入是什么；② 哪个 Harness 模块受理；③ 调用了哪段实际源码；④ 状态和事实如何变化；⑤ 事件最终怎样生成/投影；⑥ 如何用测试或 readback 证明。

## 重复讲解模块的深度递进

- **ModelStep / ToolInvocation：** Week 02 定义实体 → Week 04 解释首次模型/工具循环 → Week 07 深挖多工具并发与结果提交 → Week 11 解释事件关联 → Week 12 验证轨迹不变量 → Week 14 全链复核。
- **Run / Storage：** Week 01 全景 → Week 03 API 事务受理 → Week 04 任务领取 → Week 06 取消/恢复 → Week 11 事件与事实 → Week 13 发布/运维。
- **Tool Registry：** Week 04 最小工具 → Week 07 执行边界 → Week 09 完整 Spec/Profile/Binding/Snapshot → Week 10 真实 Adapter 接入 → Week 14 交付验证。
- **Session / Context / Evidence：** Week 02 数据结构 → Week 05 历史和模型投影 → Week 08 多跳与来源引用 → Week 12 效果分析。
- **Events：** Week 01 认识出口 → Week 03 创建事件 → Week 04/07 形成执行事实 → Week 11 完整事件协议与 SSE → Week 14 端到端呈现。
  
## 当前可验证的工程事实与公开边界

项目权威目标为 `docs/development-pack/goal-objective.md`，原课程权威为 `docs/development-pack/sources/uploaded/course-outline.md`，已实施工程以精确源码和 `docs/implementation/D-REPORT.current.md` 及各模块证据为准。

已保存的 D7 工程报告登记 M00–M20 共 21 项原工程模块门禁、25 项 API 操作和测试结果；正式封存读回与独立审查仍依照对应证据办理。真实 Retrieval/Source/Catalog 的已批准部分读回与 B/C 外部依赖分别说明。实际模型工具路线属于 T2 行为证据；人工 Gold 的实际完成数、语义质量和成本结论以正式验收记录为准。

最终系列将业务效果、Harness 运行机制、外部资源准入和交付发布四种结论独立呈现，确保每周能复现并能据实更新。

## 公开归档位置

本路线位于 `assets/insurance-agent-weekly/ROADMAP.md`。每次分享继续使用实际会议日期目录 `YYYY-MM-DD/`，并更新目录 `README.md`；Week 编号作为学习任务身份，会议日期作为归档身份。原有 Week 02 材料保留其 Git 历史，后续深入的 Harness 主体可在同一文件中更新。
