# 从一次知识查询认识 Agent Harness｜内容大纲

从一个保险条款查询进入基本模型请求、固定 RAG、Tool Calling 和 Agent Loop，再沿实际 Harness 源码完成 HTTP 入口、队列、模型、工具、持久事实到 SSE 的纵向阅读。概念先展示数据形状和执行方式，之后讨论区别、职责和设计取舍。

对应正文：[从一次知识查询认识 Agent Harness：它是什么、长什么样、怎样运行？](../01-from-rag-to-harness.md)。

## 01｜从一个具体需求开始：用户实际希望 Agent 做什么？

以等待期和意外情形查询引入实际输入。拆成等待期定义、适用条件、原文出处三类要求。展示用户能观察的受理状态、查询动作、来源和最终完成。把用户请求对应到 Turn 与 Run 的实际身份。

## 02｜最简单的 LLM 问答程序是什么、长什么样？

从一条简单 API 调用认识 LLM、Prompt、Responses API、模型输入与结构化响应。展示 Node.js SDK 的调用代码、输出条目 JSON 及 output_text 的用途。说明模型返回的是可解析对象。

## 03｜RAG 是什么？一个检索增强程序长什么样？

建立原始 PDF、解析文档、Chunk、Embedding、Evidence 的具体数据形态。画出固定检索到生成的顺序，并给出检索请求、证据行、提示词构造和最小 TypeScript 问答实现。说明固定 RAG 作为后续实验基线的意义。

## 04｜Tool 是什么？普通业务函数怎样呈现给模型？

从普通 searchDocuments 函数过渡到模型可用的 Function Tool。介绍 capabilityId、modelName、Description、参数 Schema 与返回值。展示 search_documents 的真实字段和统一业务结果协议，重点说明 partial 与来源缺口。

## 05｜Tool Calling 究竟是什么？模型说“我要查”时发回了什么？

展示一次 function_call 对象及 arguments 的字符串形式，解释 callId 如何连接 function_call_result。按“模型给出意图—SDK 执行函数—结果返回模型”说明调用链，区分规范化 SDK 条目与供应商网络响应。

## 06｜Agent 是什么？Agent Loop 在运行时究竟长什么样？

展示 Agent、Runner、Model、Session 和工具对象在 SDK 中的装配。用三次模型调用、两次工具动作的合成时序解释 Agent Loop，以及循环何时继续或结束。区分回合、模型调用与工具次数。

## 07｜Harness 是什么？在真正的软件项目里长什么样？

从部署和代码目录认识完整 Harness：composition、HTTP、storage、runner、registry、business 与 contracts。说明 Node.js、PostgreSQL、pg-boss 和 Agents SDK 的实际位置，展示公开 API 和默认合成运行模式的能力范围。

## 08｜最重要的执行对象有哪些？Thread、Turn、Run 长什么样？

完整展示 Thread、Turn、Run、ModelStep、ToolInvocation 与 Runtime Item 的身份和引用。使用实体关系图、合成聚合 JSON 和字段说明，重点讲同一输入的不同执行尝试、工具意图与结果的稳定关联。

## 09｜执行状态是什么？程序怎样知道任务是否已经结束？

将 Lifecycle 定义为对象状态迁移。展示 Run 的 queued、running、cancel_requested 以及多种终态，并区分 ModelStep 与 ToolInvocation 的独立状态。用事件子序列说明内部状态何时变化。

## 10｜Event 是什么？Harness 对外的一帧消息长什么样？

定义 Runtime Event 与 Event Envelope，展示 schemaVersion、eventId、threadId、seq、runId、type、payload。说明整数序号精度、完成事件引用与 content.delta。给出原始 SSE 帧，建立模型流与产品运行事件的区别。

## 11｜服务是怎样启动的？从 `start.ts` 看到真实装配过程

沿 start.ts 与 composeRuntime() 阅读实际创建顺序：配置、数据库和队列、Registry、HTTP 应用、Worker 与维护任务。提供启动命令和真实依赖装配片段，说明合成模型与真实服务资源各自验证的范围。

## 12｜一次用户输入怎样变成 Run？从 HTTP 走到 PostgreSQL 事务

展示创建 Thread、提交 Turn 的真实 HTTP 请求和 AcceptedTurn 回执。追踪 HTTP Handler、RuntimeStorage.submitTurn、PostgreSQL 事务、用户条目、Run、事件及 pg-boss Job。解释 revision、幂等键和 202 受理的准确含义。

## 13｜谁开始执行 Run？从 Worker 到两次 ModelStep

从 RunWorker 的 Job 解析、claimRun 和 Lease 领取进入 RunnerFacade。说明 RunEnvironment、PgSdkSession、RecordedModel 与 SDK Runner 的装配，再展示第一次模型输出 callId、ToolInvocation 提交和下一次 ModelStep。

## 14｜最终内容和事件究竟怎样产生？

跟踪助手条目保存、facts.finalAssistantIds、finalContentRef 与 finishRun()。展示完成事务如何更新 Turn、Run 和事件，再由 DurableEventStreams 经 HTTP/SSE 输出。用完整时序图收束前面所有模块。

## 15｜现在再比较：LLM、固定 RAG、Agent Loop 与 Harness 有什么区别？

在已出现的代码对象基础上比较 LLM、固定 RAG、Agent Loop、SDK 与 Harness。逐一澄清 ToolCall 与 Invocation、ModelStep 与 Run、202 与 completed、Session 与权威历史、来源引用与业务证据的区别。

## 16｜模型、SDK、Harness 和业务服务分别承担什么职责？

按 HTTP 控制面、RuntimeStorage、RunWorker、RunnerFacade、SDK、RecordedModel、Registry、Business Port/Adapter、DurableEventStreams 说明具体职责和输入输出。用 Boundary 的含义解释状态归属、错误处理和 Continuation Decision 的位置。

## 17｜为什么工程选择这样的技术架构？

解释 TypeScript/Fastify 的接口与运行时校验、Agents SDK 单 Agent、PostgreSQL 与 pg-boss 同库事务、RunSnapshot、独立 ModelStep/ToolInvocation，以及 Canonical Items/Session/Event 的组合。说明收益与数据库、版本管理和执行延迟方面的工程成本。

## 18｜实际动手：怎样验证一条从请求入口到运行事件的链路？

实践 A：使用本地 SyntheticControlModel 经 HTTP 验证 Thread/Turn/Run、Worker、ModelStep 和事件。实践 B：ScriptedModel 形成两次 ModelStep、一次工具结果回流。补充幂等重放、SQL 反查、异常定位和实际测试入口；按各模式的证据范围陈述结果。

## 19｜重新理解一句用户请求背后的软件系统

把问题重新放回 HTTP 受理、队列、Runner、模型、工具、Session、最终内容和事件全链。给出模块级排障入口和开发者应能独立推导的执行关系，连接后续数据契约与运行事实专题。

## 实践、图示与资料

以固定 RAG 与 Agent Loop 对照、执行实体关系、生命周期状态图、启动与 HTTP 事务链、完整端到端时序为核心图示。使用 JSON 和源码摘编展示真实对象；用 HTML `<details>` 展开完整证据及执行轨迹。实践分为默认合成服务生命周期、受控模型工具循环和幂等重放，再用授权测试数据库与 HTTP/SSE 交叉验证。

英文技术名词第一次出现时给出简短中文定义。采用清晰、简短的技术句子，优先解释“是什么、长什么样、怎样工作”，在正文后段讨论职责和选型。教学样本与历史工程测试采用不同的证据口径。

## 源码与依据

阅读基准为 Insurance Agent Runtime `c66112dfd8676b47b143dc2e26739278cf4b0dde`。主要对应 `apps/runtime/src/composition/start.ts`、`runtime.ts`、`transport/http/app.ts`、`storage/store.ts`、`runner/worker.ts`、`facade.ts`、`model.ts`、`tools.ts`、`session.ts`、`transport/http/sse.ts`、`packages/contracts/src/tools.ts` 和 `events.ts`。实践测试参考 `tests/control/`、`tests/runner/` 和既有 Runner 验证资料。
