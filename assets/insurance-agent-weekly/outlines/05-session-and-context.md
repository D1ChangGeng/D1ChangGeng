# 会话历史、工具结果与上下文投影｜内容大纲

用“首次检索只找到等待期定义，Agent 需要继续核对例外条件”引出主线：下一次模型请求从哪里取得已经获得的证据？将问题落到权威条目、SDK Session 与 ModelRequest 三层。

对应正文：[为什么 Agent 能够继续查：会话历史、工具结果与上下文投影](../05-session-and-context.md)。

## 01｜第二次查询为什么需要知道第一次的结果？

展示首次问题、工具调用、实际返回和第二次请求。解释证据缺口是继续查询的业务输入，而不是执行器自动推断的隐藏状态。

## 02｜Context、Session 与 Memory 分别是什么？

先展示本地 RunEnvironment、SDK Session 和模型请求的对象形状。区分持久历史、当前模型输入与应用额外记忆能力，避免用一个“上下文”覆盖所有状态。

## 03｜Canonical Item 怎样成为历史基础？

说明用户、助手、工具调用和结果的条目身份、状态与关联。介绍相同文本不同发生身份，以及已提交条目与流式条目的区别。

## 04｜PgSdkSession 的存储和接口长什么样？

展示 sdk_session_views 的 runId、itemRefs、baseRunId 与 version。解释 SDK 看到的 sessionId 是 Thread ID，而实际引用视图按 Run 保存。

## 05｜新回合怎样选择已接受的历史？

读取 initialSessionState、sessionItemEntries 与检查点引用选择。说明已完成前序回合、当前输入、失败尝试和恢复来源分别如何进入视图。

## 06｜getItems 怎样把引用还原成 ModelItem？

展示有序 ID 读取、committed 检查、Schema 解析和 WeakMap。解释 limit 只是读取范围，不是数据库删除。

## 07｜addItems 怎样防止重复输入和事实被改写？

沿已知对象、当前用户输入、恢复待持久化后缀和新事实索引四条路径讲解。分析内容摘要、发生顺序与 itemId 的角色。

## 08｜popItem 与 clearSession 实际修改什么？

用数组前后对比说明 Session 是引用视图。区别会话视图变化、历史内容删除、用户回合编辑和 Fork 等产品操作。

## 09｜工具结果何时进入下一次模型请求？

展示结果提交、事实索引、Session 追加和模型输入的实际时序。说明 pending 状态与失败提交不能被当作已获得证据。

## 10｜上下文预算到底限制什么？

读取 maxHistoryTurns、maxHistoryBytes、maxEvidenceBytes、maxRequestBytes。解释 UTF-8 JSON 字节与 token、字符数的区别，使用中文和 emoji 示例。

## 11｜projectHistoricalTurns 如何选择历史后缀？

逐步解释完整历史前缀匹配、当前回合保护、完整回合分组、最近连续后缀和独立克隆。分析无法匹配时保留输入的安全取舍。

## 12｜保护后的输入仍超预算怎么办？

展示 EvidenceBudget 与 RequestBudget 错误、观测单位和边界行为。说明程序没有实现自动摘要，后续扩展应明确压缩来源和恢复影响。

## 13｜持续补查怎样形成可观察的证据变化？

用两次检索与一次来源页读取展示新增证据、重复查询与结束选择。区分模型自主策略与 Harness 提供的输入、预算和工具权限。

## 14｜恢复为什么必须保留会话引用顺序？

解释 init 与 pre-init checkpoint 共用历史种子，恢复后未写入后缀的身份复用，以及配置兼容检查。明确当前最终 Session 不能直接证明每一次历史模型请求的完整输入。

## 15｜怎样测试历史、裁剪与连续执行？

安排相同文本不同 ID、改写已知对象拒绝、工具结果回流、clear 后恢复、完整回合裁剪与当前回合保护的测试。提供可运行的历史选择小例子与真实集成测试入口。

## 16｜把连续性落到事实和输入视图

从原始任务收束：模型能够继续行动，是因为上一轮结果准确进入下一次输入。总结 Canonical、Session 与 ModelRequest 的各自责任，连接预算、取消和恢复专题。

## 实现与展示依据

以源码 `c66112dfd8676b47b143dc2e26739278cf4b0dde` 的 `runner/session.ts`、`context-budget.ts`、`items.ts`、`model.ts`、`storage/store.ts` 为依据。配合 `docs/implementation/context-budget-preparation.md`、`recovery-boundaries-fix.md` 与相关测试。

正文使用三层数据图、历史选择图、工具结果回流时序和裁剪前后对照。HTML 折叠区展示完整历史样本及错误案例。术语首次解释，示例不混用当前会话视图和逐次模型请求快照。
