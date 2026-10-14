# Tool Registry、能力版本与运行快照｜内容大纲

用三种查询切入：解释条款、查询精确字段、枚举产品。模型需要选择不同工具，服务端需要提供清晰的动作空间和执行约束。本篇围绕真实 Registry 的对象、构造、发现、快照、绑定和调用展开。

对应正文：[让模型自己选工具：Tool Registry、能力版本与运行快照](../09-tool-registry-and-snapshots.md)。

## 01｜为什么不同问题需要不同工具？

展示文本检索、事实查询和目录分页的输入输出差异。说明模型可以自主选择，而服务端定义合法能力集合。

## 02｜Tool Registry 是什么？它在代码里长什么样？

展示 ToolRegistry 类的输入和对外方法。区分能力目录、执行后端和运行工具对象，不把 Registry 等同于一个工具名称数组。

## 03｜ToolSpec 包含什么？

逐字段说明 capabilityId、modelName、version、effect、scopes、bindingKey、handlerKey、Schema 引用、时限和大小约束。对照真实搜索能力。

## 04｜怎样设计模型能理解的工具描述？

从业务意图、参数边界、结果范围和后续动作解释。展示 content 与业务参数，保持名称和实际契约一致。

## 05｜ToolProfile 怎样选择能力集合？

说明 profileId、capabilityIds、allowedDataModes 与当前主体过滤。区分 Agent Profile、工具 Profile、模型 Profile 的不同身份。

## 06｜BackendBinding 和就绪状态是什么？

展示本地 invoke 闭包与 BackendDescriptor。区分 enabled、schema_verified、live_verified、pending_resource，说明配置存在不等于已经真实读回。

## 07｜Registry 构造时为什么校验并冻结？

读取重复能力、模型名、绑定键、Profile 引用和规格契约校验。说明描述深冻结、函数绑定捕获与运行时动态变更的边界。

## 08｜工具发现与工具执行有什么不同？

展示 listTools/getTool/currentTools 的返回和权限。目录可以描述待资源能力，但创建真实运行集合需要满足准入规则。

## 09｜RunSnapshot 怎样固定本次动作空间？

逐步读取 createSnapshot、toolset、schemaHash、policyHash 和 bindingVersions。展示快照内容摘要与后端版本分别保存，精确说明字段语义。

## 10｜bindingsFor 怎样从快照构造 SDK 工具？

核对原主体范围、工具集合摘要、策略和能力版本，再生成参数、结果、执行函数与来源提取器。展示不匹配时的明确拒绝。

## 11｜为什么快照之外仍要检查当前授权？

读取 currentPrincipal 和 intersectScopes。用 exact、prefix wildcard 与全局 wildcard 样本解释权限交集，只允许收紧原范围。

## 12｜一次调用怎样经过 Registry.invoke？

按 deadline、当前权限、后端就绪、业务参数解析、invoke、结果和来源校验走完整链。解释 runtime 写权限与业务只读效果的差别。

## 13｜工具版本变化怎样影响运行和恢复？

比较规格、Schema、后端修订、数据集和策略变化。说明新 Run 与旧快照的关系，以及不应静默切换依赖的原因。

## 14｜六类业务能力如何组织成实际工具集？

以检索、来源、保司、产品、事实和优惠为例，展示能力选择、输入范围和典型结果。注意真实数据支持按资源证据单独判断。

## 15｜如何测试发现、权限、快照与输出一致性？

提供权限交集可运行示例，目录和快照的正反例，以及真实 Registry 测试入口。区分模型选工具质量和机制正确性。

## 16｜怎样演进一个稳定的工具系统？

从新增能力、后端替换、Profile 调整和写效果准入讨论扩展。收束到业务 Port/Adapter 和依赖装配专题。

## 核心依据与展示

以 `c66112dfd8676b47b143dc2e26739278cf4b0dde` 的 `registry/registry.ts`、`catalog.ts`、`contracts.ts`、`backends.ts`、`output.ts`、`runner/tools.ts` 和工具契约为准。

正文安排四层对象图、发现与执行对照、快照装配图和授权收紧示例。每个英文概念先中文说明，再展示 JSON、函数或状态。明确目录读回、真实后端验证和实际模型路由是不同证据。
