# 如何证明 Harness 工作正确：协议测试、故障注入与效果评测

一个 Agent 在演示中完成了条款查询，工具调用顺序看起来合理，最终答案也带有来源。第二次运行，它换了一种查询方式，多调用一次工具，答案仍然正确。第三次运行，工具结果保存失败，模型却继续生成了答复。

前两次差异可能是模型行为的合理变化，第三次则涉及执行协议。测试如果只比较固定工具顺序，会把合理变化判错；如果只检查最终答案，又可能漏掉持久化故障。

Harness（智能体执行控制系统）的测试需要明确判断对象。本篇以 Insurance Agent Runtime 为例，把确定性工程验证、真实接口核对和模型业务效果分开组织，并用一条工具结果提交失败的案例串联全链。

## 先确定哪些行为由程序负责

运行系统负责受理命令、管理身份、执行授权、约束预算、保存结果和输出事件。这些行为应具有明确契约。例如，同一个幂等命令只产生一次受理，旧租约不能继续写入，已经取消的运行不能随后被普通成功提交覆盖。

模型负责在可用动作空间里选择下一步。它可以先查目录，也可以在身份已经明确时直接检索；同一个问题可能存在多条有效证据路径。测试需要允许这些变化，再根据结果和成本评价优劣。

因此，测试边界可以先分成四层：

| 层次 | 使用什么输入 | 验证什么 |
| --- | --- | --- |
| 工程协议 | 脚本化模型、合成或录制工具数据 | 状态、事务、权限、恢复和事件 |
| 真实接口 | 已授权的小规模实际请求 | 参数、响应、分页、版本和来源映射 |
| 模型行为 | 真实模型与声明明确的业务数据 | 工具选择、补查、停止和预算使用 |
| 业务质量 | 人工核验的证据样本 | 检索覆盖与答案支持性 |

这四层共享运行事实，但结论范围不同。一个工程测试通过，说明所验证的协议成立；一次模型回答正确，也不能替代并发取消和数据库故障测试。[^evaluation]

## Mock 的究竟是哪一层

Mock（测试替身）可以替换不同边界。对 Harness 全链测试，常用的是 Scripted Model（按脚本返回输出的模型）：实现模型接口，第一次返回固定工具调用，第二次读取工具结果后返回固定内容。

工具后端可以采用 Synthetic Fixture（合成夹具）或 Recorded Response（录制响应）。关键是选择准确的替换位置。

```text
真实 HTTP 入口
  → 真实请求验证和授权
  → 真实 PostgreSQL / 队列 / Worker
  → 真实 SDK Runner
  → Scripted Model
  → 真实 ToolRegistry / ToolInvocation
  → 合成业务 Port
  → 真实结果提交
  → 真实事件表和 SSE
```

这条测试真正经过产品的执行链。若直接 mock `RunnerFacade.execute()` 返回 completed，就无法证明 SDK、工具和持久结果之间的关系。

模型协议适配器还需要另一组测试。可以向适配器提供录制的供应商报文，验证它怎样转换成规范化模型事件。使用已经规范化的 Model 接口做测试，覆盖的是适配器之后的系统，不能顺带宣称原始 HTTP 协议也已经验证。

## 一条最小全链测试该断言什么

设受控模型第一次返回 `search_documents`，工具返回合成证据，第二次模型返回最终答案。测试可以观察一组关系：一个受理 Turn（用户回合）、一个当前 Run（执行尝试）、两个 ModelStep（模型调用步骤）、一个 ToolInvocation（工具执行实例），以及最终内容引用。

更重要的是它们的关联：工具实例指向第一个模型步骤，结果条目属于当前执行，第二次模型输入包含该结果，`finalContentRef` 指向已提交助手条目，完成事件包含同一个引用。[^runner]

事件可以作为外部观察面，数据库则提供内部事实。两者一起检查，可以发现“事件说完成，状态仍运行中”或“状态已完成，引用指向不存在条目”这类错误。

下面的独立 Python 示例演示部分顺序与引用断言。数据是人为构造的测试轨迹，事件负载只保留这里需要的字段。

```python
from typing import Any


def assert_one_tool_trace(events: list[dict[str, Any]]) -> None:
    sequences = [int(event['seq']) for event in events]
    assert sequences == sorted(set(sequences)), '序号必须递增且唯一'

    def one(kind: str) -> dict[str, Any]:
        rows = [event for event in events if event['type'] == kind]
        assert len(rows) == 1, f'{kind} 需要恰好一条测试事件'
        return rows[0]

    started = one('tool_invocation.started')
    finished = one('tool_invocation.completed')
    turn_done = one('turn.completed')
    run_done = one('run.completed')

    assert started['payload']['invocationId'] == finished['payload']['invocationId']
    assert int(started['seq']) < int(finished['seq'])
    assert finished['payload']['resultItemRef']
    assert turn_done['payload']['finalContentRef'] == run_done['payload']['finalContentRef']
    assert int(finished['seq']) < int(turn_done['seq']) < int(run_done['seq'])


trace = [
    {'seq': '5', 'type': 'tool_invocation.started',
     'payload': {'invocationId': 'v1'}},
    {'seq': '6', 'type': 'tool_invocation.completed',
     'payload': {'invocationId': 'v1', 'resultItemRef': 'result1'}},
    {'seq': '10', 'type': 'turn.completed',
     'payload': {'finalContentRef': 'answer1'}},
    {'seq': '11', 'type': 'run.completed',
     'payload': {'finalContentRef': 'answer1'}},
]
assert_one_tool_trace(trace)
```

这段代码只演示一个固定剧本的断言。生产测试还要查询实际条目，检查所有权、状态和模型输入，而不能把几个手写字符串当成完整验证。

## Golden Trace 应记录必要关系

Golden Trace（预期执行轨迹）适合描述受控剧本。在单工具用例里，可以精确要求两次模型调用和一次工具执行。在并发用例里，两个工具完成顺序可能合法变化，此时更适合检查 Partial Order（必要先后关系）。

例如，每个工具的 started 必须早于自己的 completed；工具结果持久化应早于消费它的下一次模型请求；最终运行提交必须引用合法内容。两个独立工具谁先返回，可以不固定。

真实模型行为更适合验证 Invariant（始终必须成立的约束）：调用属于当前 Run，工具权限符合快照与当前主体，来源引用来自可访问证据，预算结束后没有新的未授权动作。至于查询措辞和工具数量，则作为行为指标记录。

这样，测试既能捕获实现错误，也能容纳模型的合理策略差异。

## 故障注入要发生在真正的边界上

Fault Injection（故障注入）是主动安排一个可控制的失败，以验证系统反应。工具结果提交失败是一条很有代表性的案例：

```text
模型返回工具调用
  → 工具成功返回合成结果
  → 在结果数据库提交处注入失败
  → 下一次模型请求应保持未发生
  → 执行进入错误收尾
```

为什么选这个位置？因为它刚好位于外部效果与内部事实之间。若工具返回后 SDK 已经继续，而持久化在旁边异步执行，测试会看到第二次模型请求出现，这就是执行顺序错误。

工程的 `customDataExtractor` 会等待 `finishInvocation()` 保存确切的 SDK 工具结果条目，再放行后续调用。提交失败时尝试记录 `outcome_unknown` 并向上抛错；数据库仍不可用时，最终状态需要后续监督处理。[^tools]

验证应检查“失败后没有下一次模型调用”“不存在伪造的成功结果引用”“运行终态或待收敛状态与实际存储一致”。不能只断言抛出了某个异常名称。

## 使用屏障控制并发，而不是猜测时间

取消和完成竞争需要精确安排。可靠测试可以使用 Barrier（同步屏障）：模型在提交完成前等待一个 Promise，测试先提交取消命令，再释放屏障。

```ts
// 教学化的同步屏障，不依赖固定 sleep 时长。
function deferred<T>() {
  let resolve!: (value: T) => void;
  const promise = new Promise<T>(r => { resolve = r; });
  return { promise, resolve };
}

const reachedBoundary = deferred<void>();
const allowCompletion = deferred<void>();

async function controlledModelCall() {
  reachedBoundary.resolve();
  await allowCompletion.promise;
  return syntheticResponse;
}

// 测试驱动侧：启动实际产品运行后等待模型到达边界。
await reachedBoundary.promise;
await submitCancellation();
allowCompletion.resolve();
```

其中 `syntheticResponse` 与 `submitCancellation` 由测试夹具提供。测试还应设置总超时，防止屏障永远没有到达时挂住整套回归。

这种方法比“等待 100 毫秒后取消”更稳定。固定等待依赖机器负载，可能让同一测试偶尔命中不同窗口。屏障明确指定交错顺序，才能重复验证同一个因果条件。

## 控制面也属于全链测试输入

Model Events（模型事件）是模型边界返回的输出。测试驱动器还会向真实接口提交 Control Commands / Inputs（控制命令或输入），例如取消和恢复。为了组织测试时序，有些夹具把这些外部输入统称为 Control Events（控制事件）；在这个语境下，它们是驱动系统的输入，而非数据库已经保存的运行事实。

例如，测试提交取消命令，系统受理后才产生 `run.cancel_requested` 事件。命令也可能因权限或运行状态被拒绝，此时应核对拒绝结果及状态保持，而不是期待一条取消成功事件。全链测试需要同时观察输入命令、受理结果与实际持久事实。

本实现已有的控制能力包括创建和提交、取消、显式重试、检查点恢复、完成边界 Fork、归档和删除。每一种操作都具有自己的前提。测试应以实际支持矩阵为准，不把其他产品的任意回退或运行中编辑语义自动带入。[^control]

例如，同一幂等键重放请求、两个请求争用同一 Thread、旧 revision 提交、归档后继续执行、删除后重新读取，都会影响生命周期。工程中的 HTTP 测试不仅检查返回码，还读取数据库和真实队列，确认受理数量与身份一致。

恢复测试则需要验证新 Run 与旧 Run 的关联、原成功工具结果的复用、版本不匹配，以及旧租约写入被拒绝。事件续传测试还要检查连接关闭仅影响观察者，不改变后台运行。

## 业务效果需要另一份实验设计

要评价自主补查是否有效，可以固定模型、问题集、语料和检索实现，对比一次检索基线与允许多次查询的系统。报告首次证据覆盖、最终累计覆盖、调用次数、重复查询、延迟和用量。

Ablation（消融实验）进一步只改变一个机制。例如固定首轮证据和工具配置，只改变是否允许继续查询，用来观察 continuation（继续执行能力）的贡献。

若同时升级检索器和模型，再把全部提升归因于 Harness，就无法说明真正起作用的是哪一项变化。实际业务资料还会随时间变更，因此实验清单需要记录代码、模型、工具、语料和样本版本。[^evaluation]

模型参数即使相同，也可能存在随机变化。对重要结论进行多次重复，并保留失败案例和分层结果，比只展示一次最好结果更有解释力。评分程序应基于实际执行导出，避免另写一套“看起来相同”的检索逻辑导致评测与产品脱节。

## 指标必须带着分母和未知项

AnyHit（是否命中任一相关证据）与完整需求覆盖不同。找到一页等待期说明，不一定足以回答意外伤害的适用范围。多个等价来源也需要正确表示，才能避免把合理答案判成缺失。

来源存在性与语义支持性同样分开。一个引用能读到页面，证明定位可用；页面是否支持模型的断言，还需要人工或经过校准的评审。

成本计算需要实际用量和匹配的价格版本。未知用量保留未知，录制或合成价格标记为模拟。失败尝试发生过的调用仍然需要计入，不能只统计成功答案的那一段。

这些规则让“系统提升了多少”成为一个可复核命题，而不只是一个百分比。

## 从测试结果回到工程责任

一次有效的测试报告应该能够指向源码版本、输入剧本、替换边界、运行环境、断言和实际结果。它说明哪些组件真实运行，哪些依赖被替换，以及得到的结论适用于哪个范围。

当 Harness 协议、外部接口、模型行为和业务质量分别有证据，错误才容易归因，优化也容易比较。下一篇将这些验证结果带到发布环节，讨论运行系统怎样形成可交付版本：[部署、恢复与发布证据](13-deployment-and-release.md)。

## 实现依据

[^evaluation]: Insurance Agent Runtime 源码快照 `c66112dfd8676b47b143dc2e26739278cf4b0dde` 中的 `docs/implementation/evaluation-preparation.md` 及 `tools/evaluation/`；工程测试、真实接口和业务效果按独立口径说明。
[^runner]: 同一快照：`apps/runtime/src/runner/facade.ts`、`model.ts`、`session.ts` 和 `tests/runner/`。文中轨迹是教学夹具，不是新增真实业务运行。
[^tools]: 同一快照：`apps/runtime/src/runner/tools.ts` 的结果提交边界，及 `docs/implementation/runner-preparation.md` 的故障验证记录。
[^control]: 同一快照：`tests/control/http.test.ts`、`tests/control/recovery-http.test.ts`、`tests/recovery/`、`tests/events/`。公开操作与限制以实际实现为准。
