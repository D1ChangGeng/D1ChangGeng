# Insurance Agent · 每周实践分享

本目录归档 Insurance Agent 实践项目每周分享内容，采用 **会议日期（Asia/Tokyo）** 作为稳定归档标识。分享围绕 Harness Runtime（智能体运行时与执行治理）、Tool Registry（工具注册与能力目录）、业务接口集成、工程验证与下一阶段计划展开。

## 分享归档

| 日期 | 本周主题 | 分享稿 | 工程证据整理 |
| --- | --- | --- | --- |
| **2026-10-14** | 从保险业务 Agent 到可验证的 Harness Runtime：运行时链路、工具注册与执行闭环 | [阅读 Markdown](2026-10-14/weekly-sharing.md) | [Codex 证据收集指令](2026-10-14/codex-evidence-request.md) |

## 归档约定

每次分享建立一个 `YYYY-MM-DD/` 文件夹，日期对应实际分享会日期。目录中使用以下稳定文件名：

- `weekly-sharing.md`：可直接展示的中文分享正文，包括概念、架构、实际工作、验证依据与后续计划。
- `codex-evidence-request.md`：面向 Codex 的源码与测试证据核对指令。
- `evidence/`（按需）：随后取得的脱敏工程证据与索引，记录精确源码、日期、环境和验证范围。

更新同一次分享时直接修订对应文件，通过 Git 提交历史保留版本。下一周创建新的日期目录，并在本索引增添新条目。

分享中的工程实现、真实业务读回、模型评测与最终发布状态依据各自证据独立说明。公开归档采用脱敏摘要、稳定引用和可复核的验证结果，具体授权数据与密钥保留在受控工程环境。

## 当前目录

```text
assets/insurance-agent-weekly/
├── README.md
└── 2026-10-14/
    ├── weekly-sharing.md
    └── codex-evidence-request.md
```
