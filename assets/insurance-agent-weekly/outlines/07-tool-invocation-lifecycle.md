# ToolInvocation 的完整生命周期｜内容大纲

以两款产品的多维查询为切入。模型在一次响应中请求多个工具，工具返回速度、业务状态和来源完整程度各不相同。本篇从普通业务函数进入实际工具包装器，讲透调用身份、权限、并发、结果提交和模型继续。

对应正文：[一次工具调用如何成为可靠执行：ToolInvocation 的完整生命周期](../07-tool-invocation-lifecycle.md)。

## 01｜模型同时要查两个对象，程序需要保存什么？

展示原始需求、两条工具意图、不同返回结果与预期答复。区分用户问题、模型选择和实际工具动作。

## 02｜Tool、ToolCall、ToolInvocation 和 ToolResult 长什么样？

使用一个搜索能力的契约、调用 JSON、执行记录和结果对象建立直观认识。解释四者各自的身份和时间范围。

## 03｜一个 ToolBinding 如何变成 SDK FunctionTool？

读取 createRunTools、SDK tool()、参数 Schema 和执行闭包。说明工具名字唯一性、effect、版本与 content 字段检查。

## 04｜调用身份怎样关联到 ModelStep？

沿 RecordedModel.complete 建立 callSteps 映射，再到工具入口核对 runId、callId 和工具名。说明同一模型步骤可以包含多项调用。

## 05｜调用前的授权和预算怎样工作？

区分快照 scope、本地工具要求、Registry 当前授权以及资源授权。解释调用前预留、并发上限和已恢复调用拒绝重执。

## 06｜content 为什么与业务参数分开？

展示动作说明字段、实际业务参数和 RunEnvironment。说明公开动作文本不等于隐藏推理或已完成事实，机构和凭证由服务端提供。

## 07｜startInvocation 怎样建立执行记录？

展示持久字段、调用条目引用、模型步骤外键和开始事件。区别动作拒绝发生在实例创建之前或之后时的记录范围。

## 08｜后端函数执行时有哪些边界？

解释 signal 合并、执行超时、只读效果与本地取消。按工具执行位置说明业务 Adapter 的输入与返回。

## 09｜结果怎样经过 Schema、大小与来源校验？

展示 outputSchema、UTF-8 JSON 字节、sourceRefs 和 provenance。解释 SDK 宽对象包装与内部严格业务输出校验的不同职责。

## 10｜为什么结果返回后还要经过 customDataExtractor？

使用 pending 映射、SDK rawItem、canonical 转换与 itemId 展示。分析必须保存确切模型可见结果的原因。

## 11｜先提交结果再继续模型，怎样由代码保证？

逐行读取 finishInvocation、checkpoint、recordFact 和 SDK 回调完成。画出提交前后边界，提供可运行的因果顺序示例。

## 12｜执行成功与业务 partial/empty 有什么区别？

建立执行状态、业务状态、来源状态的矩阵。解释接口正常返回未知值、空列表和部分证据的合理语义。

## 13｜多工具并发怎样保留正确关联？

展示发起顺序、完成顺序和模型输入顺序。使用不同 callId 关联，不依赖数组位置猜测。解释独立并发与有依赖的多跳任务。

## 14｜结果写入失败、工具超时和取消怎样收尾？

说明 outcome_unknown、错误归一化和数据库不可用时的收敛边界。介绍当前只读工具与未来写操作恢复要求的差别。

## 15｜如何用轨迹和故障测试验证完整生命周期？

检查 ModelStep、Invocation、Call/Result Item 和事件。提供真实 Runner 测试入口、参数化 SQL，以及结果提交失败时不再调用模型的断言。

## 16｜复杂任务的可靠性如何建立在小动作上？

回到产品对比场景，说明每一项工具结果如何成为可追溯证据。连接后续多跳来源身份与 SourceRef 专题。

## 源码与实践设计

基准 `c66112dfd8676b47b143dc2e26739278cf4b0dde`。重点阅读 `runner/tools.ts`、`model.ts`、`items.ts`、`types.ts`、`registry/registry.ts` 和 `storage/store.ts` 的 Invocation 方法。

正文提供至少一个完整合法搜索输入与结果、工具状态矩阵、并发时序图、提交故障图及可运行的顺序测试。HTML 折叠区保留较长的结构和轨迹。教学示例不冒充真实业务日志，版本特定 SDK 行为引用锁定源码。
