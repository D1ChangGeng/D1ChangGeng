# 持久事件、内容引用与 SSE｜内容大纲

从一次网络断线切入：模型仍在执行，外部调用方只看见部分文字。重连后，需要知道工具是否完成、最终结果在哪里、哪些变化已经处理。围绕数据库事实到 HTTP/SSE 出口展开，不延伸到前端组件设计。

对应正文：[Harness 最终输出什么：持久事件、内容引用与 SSE](../11-durable-events-and-streaming.md)。

## 01｜观察连接断开以后，任务发生了什么？

用 Run 生命周期和观察者连接两条时间线建立问题。说明服务需要稳定状态、持久事件和重连游标。

## 02｜Runtime Event 是什么？与模型流事件有什么区别？

展示模型增量、SDK 完成、工具结果提交和产品事件的对应。区分命令输入、执行变化和日志记录。

## 03｜事件信封和 payload 长什么样？

逐字段解释 schemaVersion、eventId、threadId、turnId、runId、seq、occurredAt、type 和 payload。使用完整合法事件样本。

## 04｜事件怎样与数据库状态共同提交？

读取 emit、finishInvocation 和 finishRun。说明状态表与事件共同维护，不把存在事件表等同于完整事件溯源架构。

## 05｜Thread 序号为什么使用十进制字符串？

解释 bigint、JavaScript Number 精度、时间戳与序号的不同用途。展示超安全整数范围的比较与合法游标行为。

## 06｜内容增量怎样关联到稳定内容条目？

分析 contentRef、delta、offset 和 UTF-16 单位，区分流式、已提交、部分内容及 finalContentRef。列出最终内容不同提交路径。

## 07｜SSE 一帧到底长什么样？

展示 id/event/data、空行、注释心跳和响应头。说明 HTTP chunk 边界与 SSE 事件边界的差别。

## 08｜为什么发送200以前先读取第一批事件？

解释容量、认证、归属、游标保留范围的 preflight。中途错误只能结束流，重新连接后再获得结构化状态。

## 09｜历史重放与实时接续怎样使用同一查询？

沿 pump、cursor、batchSize 和 listEvents 展开。说明避免历史查询与消息订阅交接窗口的取舍，以及轮询负载。

## 10｜after 和 Last-Event-ID 怎样表达消费位置？

解释服务端 last written 与调用方 last processed 的不同。两个来源冲突、未来游标、单Run筛选时序号间隙分别按实际协议处理。

## 11｜Backpressure 是什么？慢读者怎样影响发送？

展示 response.write false、drain、AbortSignal 和有界批次。说明缓冲写入成功不是客户端业务处理确认。

## 12｜断线重放怎样去重和重建内容？

提供可独立运行的事件归并示例，验证大整数、重复事件、内容offset与最终引用。明确例子是内存归并器，持久消费需要自己的事务。

## 13｜事件保留窗口与410怎样恢复？

分析 earliestEventSeq、lastEventSeq、快照路径与 repeatable-read。说明当前资源快照和事件游标需要一致连接，不能分别读取后随意跳过历史。

## 14｜权限撤回、删除和限流如何影响长连接？

读取每批重新认证和资源归属检查。区别当前批次已读内容与下一次检查。说明最大连接数是当前实例配置。

## 15｜观察者和执行器怎样分别关闭？

解释 close/error清理、服务关闭destroy慢读socket、运行取消仍使用控制面。核对实际关闭顺序和计时器释放。

## 16｜如何用本地客户端和数据库核对事件？

给出curl/只读SQL与断线重连步骤，检查事件type、seq、resultItemRef和finalContentRef。说明观察窗口结束不修改Run预算。

## 17｜怎样测试重放、并发和保留窗口竞争？

建立协议、背压、授权、保留清理和最终内容一致性的测试矩阵。区分已有测试、建议故障实验和实际执行证据。

## 18｜稳定事件出口怎样成为产品契约？

汇总模型、工具、存储、事件和传输责任，连接 Harness 协议测试专题。

## 依据与展示

源码基准 `c66112dfd8676b47b143dc2e26739278cf4b0dde`。重点阅读 `transport/http/sse.ts`、`app.ts`、`packages/contracts/src/events.ts`、`storage/store.ts` 的事件读取与提交，以及 `runner/model.ts`。

正文使用事件信封JSON、SSE原始帧、数据库提交时序、重连图和背压示意。HTML折叠区展示完整重放记录和异常场景。算法示例采用本地合成事件，并明确消费方确认与网络发送的不同保证。
