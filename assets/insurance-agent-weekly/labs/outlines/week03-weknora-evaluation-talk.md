# 从一份保险问答标注认识 RAG 评估：WeKnora、错误归因与 Agent Harness｜内容大纲

以第二周已经完成的保险问答 XLSX 标注为起点，沿一条真实问题的「提问—检索—回答—人工评价」记录，学习 WeKnora 的知识问答与评估机制；再以真正执行的 API 结果理解指标，把逐题错误定位到资料、检索、生成或 Agent 决策，最后回到 D 方向已建立的 Harness 运行链和第四周最小检索循环。

这是一份供审阅的**专项技术分享大纲**，承接 [第三周实验任务](../week03-weknora-evaluation-and-attribution.md)，使用 [外部研究与源码依据](../research/week03-weknora-evaluation-source-notes.md) 完善技术解释。现场分享约 10–12 分钟；Markdown 正文可保留较充分的技术示例、源码和折叠证据，供分享前后阅读。实际分数、案例和截图待 Codex 私有实验证据包到达后引用。

## 01｜上周的标注记录是什么、长什么样？

从上周 HTML 导出的 Excel 工作簿中选择一条真实咨询问题。先展示用户原问、模型返回的答案、检索参考 Chunk 及人工标注各是什么，怎样通过题目 ID、原文文件与页码彼此关联。用一张真实字段示例图说明我们已经有了可供评测和诊断的历史记录，再指出本周需要增加的是自动化的运行与比较方式。

## 02｜WeKnora 是什么？一次 RAG 问答究竟怎样发生？

先展示 WeKnora 作为知识问答系统的基本形状：PDF 或其他文档被解析、切块、建立向量与关键词索引，用户问题经过检索、重排、模型生成得到回答。给出典型问题、检索结果对象和回答示例。随后区分快速回答（固定 RAG 管道）与模型可反复选择工具的 Agent 模式，为后面理解评估对象和 D 方向改造建立基础。

## 03｜Evaluation 是什么？一次真实评估请求怎样执行？

以一个实际请求为主线，展示 API 请求体、所需数据结构、返回的任务 ID 和任务状态，再追踪后端 Handler、Dataset Service、Evaluation Service 到 RAG 执行与 Metric Hook。解释五个 Parquet 文件分别表示问题、语料、参考答案与关联关系；明确这些文件是 WeKnora 的接口输入要求。结合真实日志描述评估知识库如何建立、题目如何执行、分数如何汇总。区分官方样例成功与自有保险样本运行的实际证据。

## 04｜指标是什么、怎么算、真实结果意味着什么？

先让读者看到真实响应的检索与生成指标字段，再用简短的实际排序示例解释 Recall、Precision、MRR、NDCG 的含义，用两种等义不同写法说明 BLEU/ROUGE 反映文本重合的能力范围。通过源码核对原生评估的语料导入、段落 ID 映射和分数平均规则，说明分数的统计单位、样本分母与输入匹配方式。给出本轮真实指标表，并将需要额外核验的实现口径作为工程研究点。

## 05｜已有保险标注怎样接入评估，得到的又是哪一种分数？

沿实际 XLSX 数据适配流程，展示如何读取已有问题、原系统回答、参考 Chunk 和人工判断，并选择这些字段能够支持的评分方法。将 WeKnora 自己重新运行的 RAG 结果，与对原保险系统**历史输出**的评测分别说明；在有相关性标签时计算检索覆盖和排名，在有实际回答与上下文时观察 Faithfulness、Answer Relevancy 等维度，在输入不足时保留明确状态。用一张单题表连接自动评分与原人工判断，形成下一章的诊断起点。

## 06｜一个问题为什么答错？怎样从分数追到具体责任？

选择一条已有人工标注、Chunk 和原始 PDF 线索的代表性案例，按「源文档是否具备正确事实—解析与切块是否完整—检索/重排是否取得所需证据—是否还需要其他查询—生成回答是否正确使用证据」逐层追踪。用原始检索排序、具体来源和自动评价解释 A/B/C/D、生成各层的责任，并保留共同原因和可验证的推断。说明“相关”“忠实于检索内容”和“保险业务事实正确”分别需要什么证据。

## 07｜固定检索为什么有时不够？怎样验证 Agent 补充查询的价值？

基于第六章中证据确实存在但首次查询尚不完整的情况，展示固定 RAG 的「原问题检索一次→生成」与自主 Agent 的「检索→识别缺口→改写或拆解查询→补充检索→再判断」两条路径。解释模型负责选择下一次动作，而检索器和事实资料保持各自职责。设计一次受控对照：固定语料和检索配置，比较首次与累计证据覆盖、查询次数、重复率、时延和成本；结果只报告已实际执行和验证的部分。

## 08｜已开发的 Harness 怎样承接这条链路？第四周从哪里开始？

从一个包含补充检索的保险问题回到具体工程：用户请求通过控制面成为 Thread、Turn 和 Run；模型调用形成 ModelStep；模型选择 Tool Registry 中的检索能力，工具执行形成 ToolInvocation，证据和最终内容拥有可追踪引用。展示现有 D 工程 `tools/evaluation/` 对固定检索、导出、评分与版本记录的支持，以及自主 Agent 运行后可以增加的轨迹评估。将这段讲述与已发布 [第02篇·数据契约与证据评测](../../02-contracts-and-runtime-facts.md)、[第03篇·请求受理与事务](../../03-request-admission-and-transactions.md)、[第12篇·Harness 测试与评测](../../12-testing-and-evaluation.md)相连接。以本周最明确的案例、数字和第四周最小 Agent 检索循环的验证目标收束。

## 实践与图示设计

以**一条真实案例**贯穿正文：上周的提问/回答/检索/标注 → WeKnora 机制图 → 实际 API 请求、运行状态与评分 → 历史保险系统评分对照 → 逐题错误归因 → 固定 RAG 与 Agent 循环对比 → 现有 Harness 的运行数据链。另选一条具有不同责任归因的案例作为对照。

核心展示包括：脱敏 XLSX/HTML 单题截图、RAG 与 Evaluation 双链路 Mermaid、五文件数据结构与请求示例、真实指标表、API/终端实测截图、错误定位的证据路径、Agent Loop 时序和 Harness 的 Thread/Turn/Run—ModelStep—ToolInvocation 关联图。

正文采用已有14篇系列的 GitHub Markdown 教学风格：首次术语中英双释，代码和 JSON 先展示具体数据形状，必要细节放进 HTML `<details>` 展开区。学术材料帮助解释评估方法和边界，实际实验结论始终由原始 API、XLSX、Chunk、源码和运行截图支撑。分享内容中的真实结果、教学示例与源码推断使用不同证据口径。

## 源码与依据

WeKnora 官方基准：`Tencent/WeKnora` 当前核查源码快照 `9224b1f40313eca00f6e5441b83174eda59a4f44`（最终正文需对照 Codex 实际运行版本）。重点读取：

- `website-docs/03-features/15-evaluation.md`、`05-retrieval-engines.md`、`07-agent.md`、`16-observability.md`
- `internal/application/service/evaluation.go`、`dataset.go`、`metric_hook.go`、`metric/precision.go`、`metric/mrr.go`
- `internal/handler/evaluation.go`、`internal/agent/engine.go`

Ragas、DeepEval 的具体指标和参数，以选定版本的官方文档与实际调用为准。理论依据重点参考 [RAGAS](https://arxiv.org/abs/2309.15217)、[ARES](https://aclanthology.org/2024.naacl-long.20/)、[SoK: Agentic RAG](https://arxiv.org/abs/2603.07379)、[RAGCap-Bench](https://arxiv.org/abs/2510.13910)、[InsQABench](https://arxiv.org/abs/2501.10943) 和 [INSEva](https://arxiv.org/abs/2509.04455)。

Insurance Agent D 的实际代码与验证记录待随私有证据 ZIP 复核，初始读取 `tools/evaluation/`、`docs/implementation/evaluation-preparation.md`、运行时 Runner、Tool Registry、Evidence、Session/ModelStep/ToolInvocation 相关实现及测试。正文将真实运行、历史证明、合成测试和进一步推断分别说明。
