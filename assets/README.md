# 资料索引

## Harness 工程实践

从一次条款查询出发，沿真实 TypeScript 工程理解请求受理、任务调度、模型循环、工具系统、持久状态、恢复、事件协议与工程验证。14 篇文章按阅读顺序编号，包含源码分析、合成案例和代码示例。

**[从第一篇开始阅读](insurance-agent-weekly/01-from-rag-to-harness.md)** · [浏览全部文章](insurance-agent-weekly/)

| 序号 | 文章 |
| --- | --- |
| 01 | [从一次知识查询认识 Agent Harness：它是什么、长什么样、怎样运行？](insurance-agent-weekly/01-from-rag-to-harness.md) |
| 02 | [一次 Agent 回答怎样才能被核验：数据契约、运行事实与证据评测](insurance-agent-weekly/02-contracts-and-runtime-facts.md) |
| 03 | [用户点击发送之后：请求受理、幂等与执行事务](insurance-agent-weekly/03-request-admission-and-transactions.md) |
| 04 | [从排队任务到模型循环：Worker、执行租约与 RunnerFacade](insurance-agent-weekly/04-worker-and-model-loop.md) |
| 05 | [为什么 Agent 能够继续查：会话历史、工具结果与上下文投影](insurance-agent-weekly/05-session-and-context.md) |
| 06 | [让自主执行保持有界：预算、取消与检查点恢复](insurance-agent-weekly/06-budgets-cancellation-recovery.md) |
| 07 | [一次工具调用如何成为可靠执行：ToolInvocation 的完整生命周期](insurance-agent-weekly/07-tool-invocation-lifecycle.md) |
| 08 | [从检索片段到原始条款：证据身份、来源引用与多跳查询](insurance-agent-weekly/08-evidence-and-source-references.md) |
| 09 | [让模型自己选工具：Tool Registry、能力版本与运行快照](insurance-agent-weekly/09-tool-registry-and-snapshots.md) |
| 10 | [把工具接到真实业务系统：依赖装配、Port、Adapter 与数据模式](insurance-agent-weekly/10-composition-and-business-adapters.md) |
| 11 | [Harness 最终输出什么：持久事件、内容引用与 SSE](insurance-agent-weekly/11-durable-events-and-streaming.md) |
| 12 | [如何证明 Harness 工作正确：协议测试、故障注入与效果评测](insurance-agent-weekly/12-testing-and-evaluation.md) |
| 13 | [把 Harness 交付出去：部署生命周期、数据库恢复与版本证据](insurance-agent-weekly/13-deployment-and-release.md) |
| 14 | [一条请求穿过整个 Harness：从条款查询到持久事件](insurance-agent-weekly/14-end-to-end-harness.md) |

## 基于 Harness 开发完整 Agent 产品

从通用 Harness API 出发，组织产品行为、执行架构、双端接入、恢复与安全，以及模块开发顺序和验收。

**[阅读软件开发指南](harness-product-development/README.md)**

[完整指南](harness-product-development/guide.md) · [API 契约参考](harness-product-development/api-contract.md) · [开发路线与验收矩阵](harness-product-development/roadmap.md) · [技术依据](harness-product-development/references.md)

## 低噪声 Agent 产品开发

围绕 AI 辅助开发中的系统理解、需求表达和可验证交付，整理为两篇主文与两份实践资料。

**[阅读系列导读](low-noise-agent-development/README.md)**

| 资料 | 主要用途 |
| --- | --- |
| [Agent 产品开发的必备知识](low-noise-agent-development/01-engineering-foundations.md) | 掌握应用宿主、通信、展示、执行、恢复与权限的关键架构判断 |
| [低噪声 AI 开发的元方法论](low-noise-agent-development/02-methodology.md) | 建立产品心智到工程模型的映射，形成最小充分的开发描述 |
| [六个跨领域案例](low-noise-agent-development/03-casebook.md) | 通过 Agent、认证支付和文件解析练习迁移方法 |
| [开发协作工具包](low-noise-agent-development/04-playbook.md) | 复制使用需求、调查、实施、变更和验收模板 |
| [技术依据与适用边界](low-noise-agent-development/05-references.md) | 查阅第一方资料及本系列的建模约定 |

## Harness 可用性测试

**[工业视角下的 Harness 可用性测试：系统分析、架构设计与技术实现](harness-testing/harness-testing-system-design.md)** · [PDF](harness-testing/harness-testing-system-design.pdf)

## 文章

[理解 Jev 模型](../articles/jev-system-one-model.md)
