# Worker、执行租约与模型循环｜内容大纲

以一条已返回 `202` 的条款查询为入口。数据库已有 queued Run，模型却尚未开始。本篇解释后台由谁领取、何时有权执行，以及一次模型响应怎样推进为工具调用和最终答案。

对应正文：[从排队任务到模型循环：Worker、执行租约与 RunnerFacade](../04-worker-and-model-loop.md)。

## 01｜接口受理以后，任务在哪里等待？

展示 Run、Job 和 HTTP 回执的不同身份。给出一个排队状态样本，建立从提交函数退出到后台继续工作的具体认识。

## 02｜Worker 是什么？它与进程、队列、Agent 有什么关系？

展示 `RunWorker` 对象、ownerId、队列回调和活动 Promise 集合。解释逻辑执行器与物理进程的差别，同进程部署仍有明确生命周期。

## 03｜pg-boss 怎样交付任务？

逐项读取 batchSize、localConcurrency、轮询间隔和 Job payload。说明这份源码每个 Worker 的本地并发为1，不能用模型工具并发参数解释整个服务吞吐。

## 04｜claimRun 怎样从 queued 取得执行资格？

追踪状态、当前授权、配置身份、数据库锁和租约建立。解释收到 Job 与真正允许执行的区别，以及领取前已取消任务的处理。

## 05｜Lease、Owner、Fence 和过期时间是什么？

展示租约字段、数据库时钟校验和心跳。用旧执行者晚到的写入说明 fencing 的用途。限定该实现对失效运行的收敛与新 Run 恢复策略。

## 06｜RunnerFacade 为什么接收 ClaimedRun？

说明门面的输入结构、依赖工厂和本次 RunEnvironment。区分本地服务端依赖、SDK context 与模型可见输入。

## 07｜Agent、Runner、Model 与 Session 怎样装配？

沿 `RunnerFacade.execute()` 展示模型工厂、工具绑定、会话、RecordedModel 和 SDK 参数。保留锁定版本信息，具体配置直接来自源码。

## 08｜ModelStep 在什么时候创建和完成？

深入 RecordedModel.begin、Provider 请求、complete 和 failed。区分产品模型调用尝试、实际传输请求、供应商重试和用量计数。

## 09｜流式响应怎样转为规范化内容条目？

展示 output_text_delta、response_done、稳定内容 ID、部分内容和完成状态。解释 SDK 流协议与产品 Runtime Event 的关系。

## 10｜模型提出 function_call 后发生什么？

展示调用意图、callId、模型步骤关联、ToolInvocation、结果提交和下一次模型输入。ModelStep 在模型响应结束时完成，工具独立记录执行。

## 11｜为什么需要同时消费 stream 并等待 completed？

解释异步迭代器、完成 Promise、即时错误监听和最终输出检查。用受控失败说明只看增量事件不能认定 Run 已完成。

## 12｜finishRun 怎样确定最终成功？

跟踪 finalAssistantIds、finalContentRef、当前取消和租约检查、终态事务。解释模型文本出现与产品成功成立之间的提交边界。

## 13｜错误、取消和停机怎样经过同一条运行链？

展示 failureCode、错误归一化、取消/中断/失败映射。读取 Worker.stop 与 Facade.stop 的职责，说明监督程序接管租约丢失后的状态。

## 14｜如何验证一条最小模型—工具—模型链？

构造受控 ScriptedModel 与合成业务工具，真实运行 SDK 和存储。展示两个 ModelStep、一个 Invocation、最终引用及事件的断言。补充重复交付、失败响应与流中断实验。

## 15｜怎样定位慢任务、重复动作和卡住的 Run？

将故障现象映射到 Job、claim、lease、ModelStep、Invocation 和终态。用少量只读 SQL 与测试入口收束，并引出下一篇 Session 和上下文。

## 图示与实践

安排队列交付图、租约时序、SDK 装配图和受控执行时序。折叠区域展示完整 ModelRequest/响应摘录与异常轨迹。提供可运行的租约资格示例，明确它只演示规则；真实分布式保证由 PostgreSQL 集成测试验证。

## 实现依据

源码快照：`c66112dfd8676b47b143dc2e26739278cf4b0dde`。

主要阅读 `apps/runtime/src/runner/worker.ts`、`facade.ts`、`model.ts`、`tools.ts`、`types.ts`，以及 `storage/store.ts` 的领取、心跳、租约与步骤提交方法。测试依据为 `tests/runner/` 和 `docs/implementation/runner-preparation.md`。

正文的术语先给中文定义，再展示实际对象和动作。主线只到服务端事实和事件输出；历史测试、合成轨迹与当轮实际执行结果分别标注。
