# Codex 指令：补齐 2026-10-14 Insurance Agent D 方向分享的最新源码证据

你是 Insurance Agent D 方向工程的证据整理者。本次任务为**只读审计与周分享资料导出**。请在已有项目源码工作区执行，目标是输出一份可提供给 ChatGPT 的脱敏 ZIP，用于核对 2026-10-14 分享材料和更新工程数字。

## 目标和边界

1. 优先核对仓库 `insurance-agent-d-product`，项目在 ACS 中的身份为 `project-insurance-agent-d`。
2. 采用实际工作树的 Git 事实：远端、分支、HEAD、tree、工作树状态、最近提交和日期。明确区分当前 HEAD `c66112d…`、历史封存候选 `fb21fa4…` 和具体测试所绑定的源码。
3. 保持仓库与业务系统原状；读取既有证据并在授权、安全且具备隔离测试环境时重跑可重复工程检查。所有真实模型、真实业务与付费 API 调用均以已有验收记录为依据。
4. 分享数据遵循项目数据授权范围。对于密钥、Token、数据库连接串、私有配置、真实个人信息和业务原文，使用状态摘要、计数、散列或脱敏示例。

## 请逐项核对

- **源码变化**：2026-10-07 至今的 Git 提交、文件变化摘要、当前 HEAD 相对最终验收候选的关键改动；分析这些变化是否影响 Runtime、Registry、协议、验收或最终封存。
- **运行时链路**：Thread/Turn/Run 受理、Worker、RunnerFacade、官方 Agents SDK、ModelStep、ToolInvocation、ToolResult、`finalContentRef`、持久事件／SSE 的实际源码路径与对应测试。
- **工具架构**：ToolSpec、Profile、BackendBinding、ToolsetSnapshot、Port/Adapter、权限交集、工具参数及结果校验、恢复版本规则；列出六项能力的 Synthetic／Recorded／Real 支持状态。
- **模块进展**：对 M00–M20（21 项工程模块）、N0–N7 治理证据、最终 bookkeeper 封存与独立验收分别提供状态及依据。
- **验证结果**：收集最新精确源码的 TypeScript、环境、Python 评测、Python 实验测试计数、命令、退出码、失败／跳过数和日志摘要。将已有历史基线与本次复核区分标注。
- **真实验证**：累积业务请求数、模型 SDK 请求数、ModelStep 对应性、M13 六 Run 自主工具轨迹、M09/M10/M11 实际业务读回，以及各项授权和来源覆盖状态；核对调用对象、版本、单位、时间与范围。
- **未决验收工作**：B/C 真实接口、来源页／原始文档 ID、上游语料版本、人工 Gold、成本、四格留出实验、最终源码读回和正式交付状态。对每项给出下一动作与明确责任依赖。

## 输出目录和文件

在仓库外创建新的报告目录，将以下文件作为 **ZIP 的最终内容**：

- `SUMMARY.md`：最近一周真正发生的工作，按“已验证 / 已实现待验收 / 外部依赖待接入”分类；每项附源码路径、提交、日期和证据。
- `ARCHITECTURE.md`：可用于讲解的真实执行链路、模块边界、Registry 数据结构、Tool 生命周期与源码映射；包含 Mermaid 图。
- `STATUS-MATRIX.csv`：模块名称、预期作用、状态、测试证据、真实资源状态、下一动作。
- `VALIDATION.md`：实际测试命令、提交身份、环境范围、通过／失败／跳过项及重要断言；对于引用的历史记录标注原始日期。
- `RESOURCE-MATRIX.md`：六项工具 Synthetic/Recorded/Real、来源授权、版本、部分覆盖与质量证据。
- `WEEKLY-SPEAKER-NOTES.md`：10–12 分钟中文汇报提纲，优先讲真实运行时和工具注册成果。
- `EVIDENCE-INDEX.md`：所有结论映射到工程路径、证据摘要和可复核的源身份。
- `logs/`：安全且脱敏的命令结果摘要和本次新增测试结果（如执行）。

## 完成判据

保证每一项进展拥有可定位的 Git／源码／验收证据，真实请求次数和测试次数分栏呈现，历史证据与当前复核清晰分开。输出 `insurance-agent-d-weekly-evidence-2026-10-14.zip`，提供 ZIP 的绝对路径、大小、SHA-256 和内容清单。完成后向我汇报该 ZIP，可由我交给 ChatGPT 做最终分享稿更新。
