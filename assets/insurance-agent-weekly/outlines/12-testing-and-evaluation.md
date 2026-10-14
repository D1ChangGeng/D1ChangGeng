# Harness 协议测试、故障注入与效果评测｜内容大纲

从一次答案正确、执行协议却可能出错的查询切入。核心工程依据为现有 tests/harness 与 tests/support/harness：使用公开 HTTP/SSE 驱动完整产品，替换模型和业务不确定性，再通过独立事实断言验证执行。

对应正文：[如何证明 Harness 工作正确：协议测试、故障注入与效果评测](../12-testing-and-evaluation.md)。

## 01｜答案正确，为什么还需要测试运行系统？

用重复受理、结果未提交即继续模型、取消后仍成功和来源错误四种场景，分开模型质量与工程协议。

## 02｜测试对象、替身和判定器分别是什么？

定义 SUT、Test Double、Fixture、Oracle 和 Invariant。展示实际产品链与被替换边界，先让读者看到测试系统的具体形状。

## 03｜Model Events 与 Control Commands 怎样共同驱动测试？

展示规范化模型条目、流式增量、工具调用、用户控制命令及受理后的运行事件。明确测试输入与已经发生的产品事件不同。

## 04｜从哪里 Mock 模型最有价值？

比较替换 Runner、SDK Model 与传输 fetch。介绍项目 Wire 测试保留生产模型工厂、SDK 和序列化解析的边界。说明模型模式与业务数据模式分离。

## 05｜真实 TestRig 怎样启动整个 Harness？

读取 createRig：临时 PostgreSQL、loopback HTTP、Registry、RunWorker、RunnerFacade、Session、公开客户端和 SSE。解释私有只读断言不向公开投影回填状态。

## 06｜脚本化模型如何检查输入和精确消费？

沿 HarnessModel、ScriptedEndpoint、modelEntry 与 immutable 展开。说明意外调用、未消费步骤、取消时合法前缀和内部断言失败保存。

## 07｜业务工具怎样在 Port 层替换？

介绍 ScriptedBusiness 的实际请求捕获、delegate、partial 和执行异常。保持 Registry、权限、预算和结果提交真实。

## 08｜最小全链路案例应该检查什么？

按 basic-loop 从公开 create/send 到两次模型调用、一次工具执行、最终导出和 SSE 对照。区分旧 Runner 单元集成与完整 TestRig。

## 09｜怎样用一项长程任务覆盖连续性？

展开 long-journey：并行目录、partial 检索、按实际 sourceRef 读取原文、字段查询、重复文本增量和下一 Turn。标明5次模型调用/5项工具动作的确定剧本。

## 10｜Golden Trace 为什么应包含因果偏序？

定义固定剧本的数量约束与并行合法变化。工具意图先提交、工具结果先于后续模型，以及终态引用一致。

## 11｜怎样构造并验证 Oracle？

沿 assertFacts、causalOrder、finalReference 和权限/租约检查。提供可运行的小型轨迹判定器，并用缺失结果和提前完成等坏样本验证它能发现错误。

## 12｜如何在结果提交处注入真实故障？

复盘 runner.test.ts 的临时数据库 CHECK 约束。说明失败前、提交后响应失败、数据库整体不可用的不同效果，严格限定隔离测试库。

## 13｜怎样控制取消与完成的竞争？

使用 Gate 与 AbortSignal 精确暂停模型或工具。介绍合作式与不合作式边界、预期脚本消费和清理，避免靠sleep猜时序。

## 14｜进程故障与租约恢复怎样测试？

依据独立Node Worker、SIGKILL、同一测试数据库和真实租约过期路径。说明抛一个异常不等价于真实进程退出。

## 15｜Stateful Testing 怎样发现命令组合错误？

读取 fast-check 的阶段/revision参考模型、合法命令前置条件、实际HTTP操作、反例缩减与seed/path重放。强调参考模型不复写产品内部算法。

## 16｜怎样定义全链路和覆盖矩阵？

说明边界覆盖、状态转移、故障、身份和控制操作矩阵。介绍 INV-01至12、25项API与真实测试结果关联；支持矩阵以当前产品为准。

## 17｜Soak、并发和资源清理验证什么？

介绍长回合、多Thread、反复SSE、内存与句柄清理。区分并发受理和当前单Worker本地执行并发1，不把合成吞吐当成真实模型性能。

## 18｜模型行为与业务效果怎样单独评测？

结合固定RAG、确定性查询重放、真实模型补查、Gold与证据覆盖。说明版本、费用、重复实验和消融，以及离线导出器没有自动产生产品Run身份的现状。

## 19｜怎样实际运行测试并保存可复现证据？

给出 test:harness、protocol、stateful、periodic、soak、replay 和独立live入口。明确blocked环境、必测零跳过、实际报告与受控路径，不执行真实付费调用作为文章副作用。

## 20｜验收报告怎样回到明确的工程责任？

说明源码、环境、替换边界、脚本消费、断言、故障和清理结果。以“输入—执行—持久事实—公开输出”收束，连接部署与发布证据。

## 实现依据与图示

源码基准为 `c66112dfd8676b47b143dc2e26739278cf4b0dde`。主要依据 `tests/harness/README.md`、`journeys.test.ts`、`stateful.test.ts`、`tests/support/harness/rig.ts`、`scripted.ts`、`oracles.ts`、`coverage.ts` 及 `tests/runner/runner.test.ts`。

正文安排替换边界图、完整测试设施图、并发join时序、提交故障图、参考状态机和证据报告结构。HTML折叠区展示请求检查、预期轨迹和坏样本。明确所列代码测试范围与实际执行范围；本地独立示例不冒充真实数据库或生产模型验证。
