# 技术依据与适用范围

> 配套《基于 Harness 开发完整 Agent 产品》 · 资料核对日期：2026-10-12
>
> [返回指南](guide.md) · [接口契约](api-contract.md) · [开发路线](roadmap.md)

本页列出用于核对协议和框架机制的官方文档、规范与项目维护者资料。正文中的架构、对象划分、API 方法、阶段顺序及验收目标是本指南的参考设计；采用者需要在具体项目中确认、实现和验证。

在线文档可能随版本更新。实施时保存框架版本、依赖锁文件、协议版本和实际行为测试，尤其关注取消、审批、检查点、流式协议、原生模块和平台差异。资料引用支持相应机制，不表示引用项目已经实现本指南的全部产品契约。

## 接口与传输规范

<a id="s01"></a>
### S01 · HTTP 请求与响应语义

[RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)

用于核对 HTTP（请求响应协议）的状态与方法语义，尤其区分任务已受理和工作已完成。文中的具体路径、命令幂等及等待规则由本指南定义。

<a id="s02"></a>
### S02 · HTTP API 错误详情

[RFC 9457 — Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)

用于错误响应结构，包括问题类型、标题、状态及说明。业务代码、资源隐藏规则和可重试判断由应用定义。

<a id="s03"></a>
### S03 · 服务器发送事件

[WHATWG HTML — Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)

用于 SSE（服务器发送事件）的格式、EventSource 接口、事件标识与重连机制。持久化、快照位置、已应用游标及补齐保证属于应用协议。

## 执行框架与上下文

<a id="s04"></a>
### S04 · OpenAI Agents SDK 运行机制

[Running Agents](https://openai.github.io/openai-agents-js/guides/running-agents/)

用于核对 SDK（软件开发工具包）中的执行循环、工具处理及运行结果。产品 Turn、RunAttempt 和 ModelStep 需要与实际 SDK 事件建立映射。

<a id="s05"></a>
### S05 · OpenAI Agents SDK 流式执行

[Streaming](https://openai.github.io/openai-agents-js/guides/streaming/)

用于核对完整事件流、审批暂停、取消、清理完成及使用运行状态继续。文本流结束不能单独作为产品任务终态依据。

<a id="s06"></a>
### S06 · OpenAI Agents SDK 人工审批与状态

[Human-in-the-loop](https://openai.github.io/openai-agents-js/guides/human-in-the-loop/)

用于核对审批中断、状态序列化、继续及可信存储要求。序列化状态可能包含上下文和审批信息；完整性、所有权和重放保护需要应用负责。

<a id="s07"></a>
### S07 · OpenAI Agents SDK 会话存储

[Sessions](https://openai.github.io/openai-agents-js/guides/sessions/)

用于核对会话接口和历史管理能力。应用、本地 SDK 与提供方会话状态之间的权威关系应在产品内确定。

<a id="s08"></a>
### S08 · OpenAI Agents SDK 上下文

[Context Management](https://openai.github.io/openai-agents-js/guides/context/)

用于区分应用本地上下文与模型可见内容。凭据、租户身份、工具上下文和检查点中的数据需要分别设计访问与保存策略。

## 展示与界面状态

<a id="s09"></a>
### S09 · AI SDK 界面流协议

[Stream Protocols](https://ai-sdk.dev/docs/ai-sdk-ui/stream-protocol)

用于核对 UIMessage（界面消息）的文本、工具、自定义数据和流结束格式。公共 Harness 的任务状态、游标和快照重建属于需要额外定义的语义。

<a id="s10"></a>
### S10 · React 状态保留

[Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state)

用于核对组件类型、位置与键对状态保留的影响。业务身份稳定和组件本地状态保留是两个相关但不同的问题。

## 持久化与可靠性

<a id="s11"></a>
### S11 · PostgreSQL 事务隔离

[Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)

用于核对并发事务的可见性、隔离及异常。具体状态转换还需条件写入、唯一约束、锁或其他并发控制。

<a id="s12"></a>
### S12 · 事务性发件箱

[AWS Prescriptive Guidance — Transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)

用于协调业务数据提交与事件投递。该模式仍需消费端处理重复消息，且不能自动保证任意外部工具副作用恰好一次。

<a id="s13"></a>
### S13 · SQLite 预写日志

[Write-Ahead Logging](https://www.sqlite.org/wal.html)

用于核对 WAL（预写日志）的读写行为、单写入者和共享存储限制。本指南将本地数据库访问集中到后台接口。

## 权限、工具与安全

<a id="s14"></a>
### S14 · 权限判定

[OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)

用于最小权限、默认拒绝、资源访问校验和授权测试。人工审批与工具执行范围是应用中的具体授权关系。

<a id="s15"></a>
### S15 · 提示词注入防护

[OWASP LLM Prompt Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html)

用于理解不可信文件、网页和工具结果影响模型行为的风险及多层防护。测试若干样本不构成全面安全保证。

<a id="s16"></a>
### S16 · MCP 工具协议

[Model Context Protocol — Tools, 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)

用于工具发现、调用和结果的协议表示。本指南核对时，官方 latest 路径指向此版本；项目采用其他版本时应核对对应规范。工具权限仍由产品执行边界落实。

<a id="s17"></a>
### S17 · Node.js 子进程

[Child Process](https://nodejs.org/api/child_process.html)

用于程序启动、标准输入输出、错误和进程结束行为。进程树清理、资源限制和强隔离需要结合实际操作系统验证。

<a id="s18"></a>
### S18 · Node.js 事件循环响应性

[Don't Block the Event Loop](https://nodejs.org/learn/asynchronous-work/dont-block-the-event-loop)

用于理解异步等待与同步计算对控制响应和吞吐的影响。任务预算与调用方等待属于应用层的独立规则。

## 桌面、终端与接口实现

<a id="s19"></a>
### S19 · Electron 进程模型

[Process Model](https://www.electronjs.org/docs/latest/tutorial/process-model)

用于主进程、渲染进程、预加载及实用进程的职责。独立常驻 Agent 后台是本指南选择的一种产品部署方式。

<a id="s20"></a>
### S20 · Electron 安全

[Security](https://www.electronjs.org/docs/latest/tutorial/security)

用于上下文隔离、进程沙箱、受限接口、导航和发送方检查。桌面界面安全与任意工具程序的操作系统隔离分别验证。

<a id="s21"></a>
### S21 · Electron 进程间通信

[Inter-Process Communication](https://www.electronjs.org/docs/latest/tutorial/ipc)

用于请求、响应、主动消息及预加载桥梁。业务接口契约保持独立于通信通道名称。

<a id="s22"></a>
### S22 · Commander.js 命令参数

[Commander.js 官方仓库](https://github.com/tj/commander.js)

用于命令、参数、帮助和错误输出的组织。文中的 `agent` 命令是拟定产品接口。

<a id="s23"></a>
### S23 · Ink 终端交互

[Ink 官方仓库](https://github.com/vadimdemedes/ink)

用于 React 风格终端组件和交互。模型执行、同步、审批和输出契约由产品客户端及 Harness 提供。

<a id="s24"></a>
### S24 · Fastify 校验与序列化

[Validation and Serialization](https://fastify.dev/docs/latest/Reference/Validation-and-Serialization/)

用于基于结构定义的请求校验与响应处理。运行时 Schema（结构约束）需要可信来源；授权、幂等和状态转换属于独立业务检查。

## 测试与可观测性

<a id="s25"></a>
### S25 · Playwright Electron 自动化

[Electron](https://playwright.dev/docs/api/class-electron)

用于桌面自动化接入。官方标注其 Electron 支持为实验性；项目需锁定组合并保留真实安装环境验收。

<a id="s26"></a>
### S26 · OpenTelemetry 调用追踪

[Traces](https://opentelemetry.io/docs/concepts/signals/traces/)

用于理解追踪、片段和跨组件关联。诊断追踪与业务权威记录有不同保留、采样和一致性目标。

## 同仓库的配套阅读

[低噪声 Agent 产品开发](../low-noise-agent-development/README.md)介绍系统理解、心智模型与工程表达的方法。

[工业视角下的 Harness 可用性测试](../harness-testing/harness-testing-system-design.md)提供更深入的测试体系材料。

采用相关资料时，应将当前项目确认的接口、状态、版本和验证结论写入自己的实施基线。
