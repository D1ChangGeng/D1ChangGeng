# 一次 Agent 回答怎样才能被核验｜内容大纲

以等待期和意外条件查询为案例，将检索证据的身份与评分要求，连接到 Harness 的运行条目、工具执行、接口契约和事件。正文先呈现 Evidence 与 Gold 的数据形状，再解释权威事实、类型校验、反向追踪和评测实践。

对应正文：[一次 Agent 回答怎样才能被核验：数据契约、运行事实与证据评测](../02-contracts-and-runtime-facts.md)。

## 01｜Agent 已经返回答案，怎样判断它找对了资料？

从等待期和意外适用条件两项断言开始，区分页面是否存在、原文是否支持断言，以及 Agent 是否实际读取该证据。提供合成回答及可核验要求表，建立业务效果、来源定位与运行事实三层判断。

## 02｜Evidence 是什么？一次检索到底返回什么？

解释 Evidence 在搜索结果中的具体字段：正文、chunkId、documentId、page、score、scoreMetric、metadataStatus、sourceRef、provenance。展示统一 Tool I/O 返回结构、原始排名与相同物理页多 Chunk 的归并关系。

## 03｜一份 PDF 为什么会产生多种身份？

从原始 PDF 到解析页、Chunk 和索引画出数据身份转换链。区分文档 ID、文档版本、物理页、印刷页和索引位置。说明切分策略变化为什么不能改变人工评分的稳定物理页定位单位。

## 04｜Gold Dataset 是什么？标准证据怎样写进数据文件？

定义 Gold Dataset、Case、labelKind 和 answerability。展示 dataset.synthetic.json 的真实字段形状与一个合成问题样本。说明 synthetic_fixture、machine_reference、human_reviewed_gold 的不同证据地位。

## 05｜一个问题有多项要求时，怎样表示“证据齐全”？

用 evidenceRequirements 与 alternatives 表示一个问题的多个子要求。解释任一备选路径成立及单条路径内所有页面齐备的逻辑，并展示跨页、跨文档和等价证据两种覆盖情况。

## 06｜一次检索只返回部分资料，工具怎样表达这个事实？

从真实 ToolResult 字段解释 status、coverage、missing、warnings 和 provenance。展示 sourceRef 未发行、documentId 缺失时的 partial 样本，并区分工具执行成功与业务证据仍不完整。

## 07｜运行事实是什么？工具返回值怎样成为可追溯数据？

沿 SDK 工具结果到 runtime_items、tool_invocations 和事件事务展示 Canonical Fact。讲清进程内函数结果、已提交 Runtime Item 与后续模型可见 ToolResult 的三种时刻，强调提交先于继续。

## 08｜Thread、Turn、Run 与 ModelStep 的契约具体包含哪些字段？

对照公开 DTO 与 PostgreSQL 表，按 Thread、Turn、Run、ModelStep、ToolInvocation 展示关键字段。分类说明对象身份、状态、版本、引用和统计用量，并提供简化聚合 JSON 和只读 SQL。

## 09｜Canonical Item 是什么？为什么一段文本还需要独立 ID？

解释 runtime_items 的 itemId、role、origin、status、canonical、digest、text、source_refs。展示相同文本不同条目身份、流式与已提交状态，并比较内部 Canonical Item 与公开 ContentItem 投影。

## 10｜ToolCall、ToolInvocation 与 ToolResult 怎样连接成一条记录？

展示 ModelStep 下多个 function_call 通过 callId、providerCallId、modelStepId、callItemRef、resultItemRef 连接到各 ToolInvocation。结合多工具并发说明结果顺序不能替代调用身份，并提供只读关联查询。

## 11｜finalContentRef 是什么？最终答案为什么只保存一个引用？

沿模型完成、facts.finalAssistantIds 与 finishRun() 解释 finalContentRef。检查 Turn、Run 和已提交助手条目指向相同内容，并说明重试时旧执行事实与当前回合投影分别保留什么。

## 12｜RunSnapshot 与 Provenance 是什么？如何复现一次执行？

展示 RunSnapshot 保存模型、SDK、工具集与策略身份，Provenance 保存数据模式、后端、数据集版本和观察时间。比较在线 Snapshot 与离线 Evaluation Manifest 的目的，说明可复现需要记录哪些环境变化。

## 13｜Runtime Event 的契约是什么？事件怎样关联执行事实？

展示 Runtime Event 信封及 tool_invocation.completed、run.completed 等类型化 payload。解释 thread 内 seq、eventId、引用和 bigint 字符串，说明事务事实如何映射到 HTTP/SSE。

## 14｜Contract 是什么？TypeScript、Zod、JSON Schema 有什么作用？

定义 Contract、TypeScript 类型、Zod 与 JSON Schema 的不同用途。用 SubmitTurn.safeParse 的合法和非法输入说明运行时校验、strictObject、nullable、optional 和类型转换边界。

## 15｜HTTP API 的输入与输出契约长什么样？

拆解 POST /v1/threads/{threadId}/turns 的输入、路径、Header、AcceptedTurn 和 ErrorEnvelope。区别 Schema 错误、版本冲突和幂等冲突，说明 202 受理与执行终态的关系。

## 16｜Tool I/O Contract 如何校验模型参数与业务结果？

展示 search_documents 的输入字段与 topK 范围，再执行合法与非法参数解析。说明 Registry 的输出 Schema、sourceRefs 与 provenance 复核，以及 SDK 宽包装层和业务严格校验的不同位置。

## 17｜OpenAPI 是什么？怎样让接口文档与代码保持一致？

从 document.ts 和 routes.ts 展示 Zod 到 JSON Schema/OpenAPI 的生成链。解释 Schema Drift、真实注册路由清单核对及 typecheck/check:contracts/check:control 的作用，演示新增必填字段的兼容影响。

## 18｜怎样从最终回答反向找到模型、工具和原始证据？

沿 Turn.finalContentRef → Assistant Runtime Item → ModelStep → Session → ToolResult → Invocation → Evidence/sourceRef 反向追踪。配套只读 SQL 和事件核对，区分执行关联、来源可读和原文语义支持。

## 19｜AnyHit、MRR、EvidenceRecall 和 RequirementCoverage 怎样计算？

解释 AnyHit@K、MRR@K、EvidenceRecall@K 与 RequirementCoverage。展示去重物理页、保留未知候选位置、备选证据路径、First/Final 汇总与多次检索累计。配可独立运行的 Python 示例和数值结果。

## 20｜动手建立完整的执行与证据验证链

实践 A：固定检索基线导出与评分。实践 B：确定性多次查询重放和累计覆盖。实践 C：基于真实 PostgreSQL/SDK Runner 的模型与工具执行断言。明确离线 exporter 不会自动产生产品 Run，分开报告协议、外部服务、模型行为和业务质量。

## 实践、图示与资料

重点安排原始 PDF→Chunk→Evidence 身份图、工具返回→持久化事实时序、Run/ModelStep/Invocation 的引用关系、事件信封与原始 SSE、契约生成链、最终结果反向追踪和多轮评分图。通过 HTML `<details>` 展示结构完整的合成证据、非法请求、运行关联与评分解释。

可复现材料包括 Zod 的合法/非法参数解析、参数化只读 SQL、独立 Python 指标计算，以及项目已实现的离线导出与评分命令。区分合成评测集、实际模型调用、业务来源与 Harness 集成测试的结论范围。

## 源码与依据

阅读基准为 Insurance Agent Runtime `c66112dfd8676b47b143dc2e26739278cf4b0dde`。主要对应 `packages/contracts/src/api-schemas.ts`、`tools.ts`、`events.ts`、`document.ts`、`routes.ts`，以及 `apps/runtime/src/storage/schema.ts`、`store.ts`、`runner/model.ts`、`tools.ts`、`session.ts`。评测依据为 `tools/evaluation/examples/dataset.synthetic.json`、`exporter.ts`、`score.py`，执行验证对应 `tests/runner/` 和相关协议测试。
