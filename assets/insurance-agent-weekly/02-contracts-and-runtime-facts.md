# 让 Agent 的行为可以被验证：契约、运行实体与证据标尺

查询系统返回了一段保险条款，答案末尾还写着“来源：第 12 页”。这看起来已经很完整。真正检查时，我们却需要继续追问：是哪份文件的第 12 页？指 PDF 的物理页还是印刷页码？这一页支持的是等待期长度，还是意外情形的适用条件？模型是否真的读取过它？

这几类问题分别涉及业务证据、执行事实和数据契约。要把 Agent 做成可以维护的系统，需要给每一种事实选择稳定的表达方式。否则，测试脚本、运行日志和人工评审虽然都在说“正确”，判断的对象却可能完全不同。

本文从一个条款核验场景出发，说明怎样让 Harness（智能体执行控制系统）拥有可验证的数据基础。工程实例采用 Insurance Agent Runtime，条款、标识和评分样本均为合成示例。

## 先把“查到了”拆成可检查的判断

设用户的问题是：“示例保障甲的等待期是多少，意外伤害是否适用同样规则？”这个问题至少有两个证据要求：一个支持等待期长度，另一个支持意外情形的适用条件。

我们可以建立一份 Gold Dataset（人工核验的评测集），记录哪些原始页面能够支持这些要求。这里的 Gold 是经过原文核验的参照，不是系统本次检索结果的别名。

```json
{
  "caseId": "SYN-Q-01",
  "question": "示例保障甲的等待期及意外情形如何规定？",
  "answerability": "answerable",
  "evidenceRequirements": [
    {
      "name": "等待期长度",
      "alternatives": [["SYN-DOC-A:12"]]
    },
    {
      "name": "意外情形适用范围",
      "alternatives": [["SYN-DOC-A:13"], ["SYN-DOC-B:4"]]
    }
  ]
}
```

这是教学化的评测结构。第二项允许两套替代来源：A 文件第 13 页，或者 B 文件第 4 页。某些判断必须把两页结合起来，便可以让同一个 alternative（备选证据路径）包含两个页面，表示二者共同支持结论。

原始 PDF 的人工核验决定这些集合。检索结果可以帮助定位候选页面，但“页面存在”和“页面支持这个要求”要分别检查。对于资料尚未核实完整的样本，使用 `unresolved`（待确认），避免把未知情况写成确定不可回答。工程评分说明采用了这类区分。[^evaluation]

## 身份比显示名称更适合做连接键

一次检索可能返回 `chunkId`（片段标识）、`documentId`（文档标识）、文件名和页码。文件名适合阅读，稳定标识适合关联。相同文件名可能对应多个修订版，片段重新切分后也可能得到新的 `chunkId`。

因此，评分单元可以归一成“机构范围、稳定文档身份、物理页码”。文档版本需要由数据集版本或明确版本字段锁定。模型引用使用的 `sourceRef`（授权来源引用）则由运行系统签发，用于在当前权限下读取来源。

这两种引用服务不同目的。人工评测需要长期可比较的证据身份；一次运行的来源读取还要接受当前主体、文档范围和有效期的校验。把它们连接起来需要服务端映射，而不是直接比较两段字符串。

还有一个容易忽略的细节：源数据缺少文档标识时，Harness 应保留这个缺口。系统可以保存已获得的正文和片段身份，但自动评分只能在定位条件充分时建立匹配。未定位的结果仍应留在原始结果序列中，不能删除它之后把后面的好结果悄悄提前。

## Harness 保存的是一组相互关联的运行对象

业务证据说明“答案依据在哪里”，运行实体说明“系统怎样取得了它”。这套实现采用以下对象：

| 对象 | 中文理解 | 最重要的关联 |
| --- | --- | --- |
| Thread | 持续对话线程 | 机构、主体、历史回合 |
| Turn | 一次正式受理的用户回合 | `userInputRef`、当前 Run、`finalContentRef` |
| Run | 完成一个回合的一次执行尝试 | 快照、租约、重试或恢复来源 |
| ModelStep | 一次进入模型边界的调用尝试 | Run、顺序、响应条目和用量 |
| ToolInvocation | 一次具体工具执行 | ModelStep、调用标识、参数、结果引用 |
| Runtime Item | 规范化运行条目 | 用户内容、模型内容、工具调用和结果 |
| Runtime Event | 运行变化的持久事件 | Thread 内序号、Run、类型、负载 |

这些对象在产品存储中保持自己的身份。[^storage]

假设一次运行的模型先调用检索工具，再根据结果回答，可以得到这样的关系：

```text
Turn U1
  userInputRef → Item I1
  currentRunId → Run R1

Run R1
  ModelStep S1 → ToolCall Item C1
  ToolInvocation V1
    modelStepId → S1
    callItemRef → C1
    resultItemRef → Item E1
  ModelStep S2 → Assistant Item A1

Turn U1.finalContentRef → A1
```

`finalContentRef` 是最终内容引用。它指向已经保存的助手条目，而不是再复制一份最终答案。这样，回合详情、历史读取和导出都有同一个内容身份可以核对。

同一句话在两次不同输入中出现，也应得到两个输入条目。对执行系统而言，重复文本与重复请求是不同问题：前者可能是用户确实又问了一次，后者需要由幂等键判断是否重放原命令。

## 契约要同时约束入口和出口

Schema（数据结构约束）决定哪些请求和结果能够进入系统。TypeScript 类型帮助开发时检查调用方式；运行时 Schema 则处理来自 HTTP、模型和外部服务的实际数据。

例如，业务调用方可以提交用户输入和已授权的 Agent Profile（运行配置名称）。机构身份、模型凭证和上游服务地址则由服务端构造。这里是一份简化的教学 Schema：

```ts
import { z } from 'zod';

const SubmitTurn = z.strictObject({
  input: z.strictObject({ text: z.string().min(1) }),
  agentProfileId: z.string().min(1),
  expectedThreadRevision: z.number().int().nonnegative(),
});
```

`strictObject` 对额外字段执行明确校验。工程的 HTTP 入口还关闭了验证器自动移除未知字段、自动转换类型和注入默认值的配置，使非法输入可以按产品契约报告，而不是先被框架改写。Fastify 官方资料将请求验证与响应序列化作为两个独立环节；项目在此基础上使用自己的权威契约生成路由结构。[^http][^fastify]

实际契约源会生成 OpenAPI（HTTP 接口描述）、JSON Schema（跨语言数据结构）和客户端类型。工程还比较真实注册的路由与声明清单，避免文档写着支持某个端点，而运行服务尚未注册它。

这种一致性检查可以放入普通回归：

```bash
corepack pnpm check:contracts
corepack pnpm check:control
```

命令针对已经安装项目锁定依赖的源码目录。它们验证生成物与实际声明是否一致；业务行为仍需由后续集成测试确认。

## 用小例子说明指标究竟量了什么

下面的 Python 示例只演示证据需求评分，不依赖模型，也不访问业务数据。候选中相同页面只计一个评分单元，缺失定位保留一个占位。

```python
from typing import Optional

Locator = tuple[str, int]

def normalize(rows: list[Optional[Locator]]) -> list[Optional[Locator]]:
    seen: set[Locator] = set()
    units: list[Optional[Locator]] = []
    for row in rows:
        if row is None:
            units.append(None)
        elif row not in seen:
            seen.add(row)
            units.append(row)
    return units


def score(
    rows: list[Optional[Locator]],
    requirements: list[list[set[Locator]]],
    k: int,
) -> dict[str, float]:
    if k < 1 or not requirements:
        raise ValueError('k 和证据需求必须有效')
    if any(not alternatives or any(not path for path in alternatives)
           for alternatives in requirements):
        raise ValueError('每项需求至少包含一条非空证据路径')

    top = normalize(rows)[:k]
    found = {row for row in top if row is not None}
    relevant = set().union(*(path for alternatives in requirements
                             for path in alternatives))
    first_rank = next((i for i, row in enumerate(top, 1)
                       if row is not None and row in relevant), None)
    completed = sum(any(path <= found for path in alternatives)
                    for alternatives in requirements)
    return {
        'AnyHit@K': float(first_rank is not None),
        'MRR@K': 0.0 if first_rank is None else 1.0 / first_rank,
        'RequirementCoverage': completed / len(requirements),
    }

requirements = [
    [{('SYN-DOC-A', 12)}],
    [{('SYN-DOC-A', 13)}, {('SYN-DOC-B', 4)}],
]

print(score([('SYN-DOC-A', 12), None, ('SYN-DOC-B', 4)], requirements, 2))
print(score([('SYN-DOC-A', 12), None, ('SYN-DOC-B', 4)], requirements, 3))
```

按该定义，两个输出的 AnyHit 都是 1，MRR 都是 1，但完整需求覆盖分别是 0.5 和 1.0。前两项表明“较早找到了相关证据”，第三项才区分“只支持一半问题”和“全部要求都有证据”。

工程评分器还处理 EvidenceRecall（证据召回比例）、多轮累计结果、等价证据路径、未知文档、跨机构数据和缺失轮次。多轮结果的累计覆盖基于实际获得证据的并集；不同搜索的原始排名仍各自保留。将每轮排名拼成一个虚构的全局排名，会改变指标的含义。[^evaluation]

## 工程测试与业务评测使用同一事实基础

有了稳定对象和契约，测试可以直接询问系统事实。

例如重复提交一个请求，应该检查回执里的 Turn ID 和 Run ID 相同，数据库只保存一个用户输入，并且队列中只有对应的一次受理任务。检查来源读取，则应同时验证授权、文档版本和页码映射。检查模型行为，需要查看模型实际选择了什么工具，以及结果是否提供了新的证据。

这形成四个互补的验证层次：

| 验证对象 | 主要输入 | 结论范围 |
| --- | --- | --- |
| Harness 协议 | 受控模型与合成工具 | 状态、权限、事务、恢复和事件是否符合契约 |
| 外部接口 | 已授权的真实调用 | 请求、响应、分页、来源映射是否匹配 |
| 模型行为 | 真实模型与明确数据模式 | 工具选择、补查、停止和预算表现 |
| 业务质量 | 人工核验样本 | 证据是否足够，回答是否获得原文支持 |

这张表也是排障路线。工具成功返回但答案错误，可能涉及业务口径或模型生成；工具结果根本没有进入下一次模型请求，则需要检查 Harness 的持久化与 Session。前者可以通过评测发现，后者应建立确定性回归。

## 一份可复核的记录需要保存哪些上下文

仅有一个分数，很难解释变化。一次评测至少需要把问题集版本、代码版本、模型配置、检索配置、工具集合和数据模式连接到原始结果。模型与工具调用的计数也要明确口径。

ModelStep 是模型边界上的调用尝试。由于记录会先于实际网络发送写入数据库，它有可能表示一个在发送前失败的尝试；底层客户端若执行自动重试，也可能产生额外请求。因此，计量严格时应把传输层的实际请求记录与 ModelStep 对照，而不是把某一个计数视作天然等价于账单调用数。

同样，`real` 表示结果声明的真实数据模式，还需要具体接口读回支持；`recorded` 表示录制回放；`synthetic` 表示合成数据。工程中这些来源信息随工具结果保存。模型是否真实调用，是另一条独立维度。

这样一来，结果报告可以说清楚：“这份分数来自哪个问题集、哪组工具结果和哪次执行”，而不只是给出一个看起来精确的百分比。

## 把正确性落实到对象之间的关系

可靠的 Agent 系统需要两把尺子：一把检查执行机制，一把检查业务证据。连接它们的是稳定身份和明确契约。

一份经过核验的来源页，通过检索结果进入 ToolInvocation，再成为模型下一次调用的输入，最终与已提交的内容关联。沿这条关系链，开发者可以定位错误，评测程序可以计算指标，维护者可以复现当时发生的事情。

接下来，第一条需要具体实现的关系是“外部请求怎样成为一个持久 Run”。[请求受理与事务控制](03-request-admission-and-transactions.md)会沿 HTTP 入口追踪这一步。

## 实现依据

[^storage]: Insurance Agent Runtime 源码快照 `c66112dfd8676b47b143dc2e26739278cf4b0dde`：`apps/runtime/src/storage/store.ts`、`apps/runtime/src/runner/model.ts`、`apps/runtime/src/runner/session.ts`。实体身份、模型边界和引用行为来自这些实现。
[^http]: 同一快照：`apps/runtime/src/transport/http/app.ts`、`packages/contracts/src/api-schemas.ts`、`packages/contracts/src/tools.ts`，以及 `tests/control/http.test.ts` 的重复受理断言。
[^evaluation]: 同一快照中的 `docs/implementation/evaluation-preparation.md`。本文评分程序是独立教学实现，只演示列出的三个指标；生产评分器的全部输入校验和多轮处理以工程代码为准。
[^fastify]: [Fastify：Validation and Serialization](https://fastify.dev/docs/latest/Reference/Validation-and-Serialization/)。框架机制与产品选择分别说明。
