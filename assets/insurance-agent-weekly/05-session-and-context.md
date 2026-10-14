# 为什么 Agent 能够继续查：会话历史、工具结果与上下文投影

第一次检索只找到等待期定义，缺少意外伤害的适用条件。系统要完成这个咨询，通常需要把已经取得的证据交给模型，再让模型决定补查什么。

这里有一个具体工程问题：下一次模型请求的输入从哪里来？如果只发送原始问题，模型无法知道前一次查到了什么；如果把数据库里的所有记录原样发送，失败尝试、重复输入和过长历史又可能混入上下文。

持续执行依赖一条稳定链路：工具结果成为持久事实，Session（会话历史适配器）选择并排列可见条目，模型请求再根据预算生成自己的输入视图。Insurance Agent Runtime 的实现把这三层明确分开。

## 模型继续执行，需要的是新增事实

Agent Loop（模型与工具交互循环）可以用一个简单关系描述：

```text
本次模型输入
  → 模型输出工具调用
  → 工具执行并保存结果
  → 结果加入下一次输入
  → 模型继续选择动作或给出答复
```

假设合成产品“示例保障甲”的第一次检索返回一个定义片段。返回内容还指出来源信息不完整。模型下一次输入需要包含原问题、调用过什么工具、实际返回内容，以及工具协议中的 `missing`（缺失项）、`warnings`（警告）和 `coverage`（覆盖状态）。

工程的 ToolResult（工具结果）已经有这些字段。它们表达本次工具执行取得了什么，以及结果能够支撑到什么范围。模型可以据此换一个查询，扩大 `topK`（最多召回条数），或使用已经签发的来源引用读取原文页。[^tool]

“具体还缺什么”比笼统的“资料不足”更有行动价值。不过，需要区分工具能确定的元数据缺口与模型对语义充分性的判断。工具知道源文档标识缺失，并不天然知道回答整个业务问题还缺哪些条件。后者需要模型结合问题和证据进行判断，随后通过新的动作表现出来。

## 三种上下文具有不同生命周期

谈 Agent 的“记忆”时，至少需要区分三个对象。

**RunEnvironment（本地执行环境）**存放主体、存储、客户端依赖、取消信号、预算和租约。这些值帮助程序执行工具，不会因为作为一个本地对象存在就自动进入模型输入。

**SDK Session（SDK 会话历史）**提供模型循环需要读取和追加的消息协议。它回答“本次运行在什么历史上继续”。

**ModelRequest（模型请求）**是某一次真正交给模型适配器的输入，包含系统指令、消息、工具定义和模型设置。它可以是完整历史的一个预算化投影。

OpenAI 的上下文说明区分了本地依赖上下文和模型可见内容；工程在此基础上，用独立的引用视图管理持久会话。[^context][^session]

```text
本地运行环境
  主体、存储、信号、预算、租约
                  │ 供本地代码使用
                  ▼
权威条目 → SDK Session 有序引用 → 本次 ModelRequest 投影
```

这样，压缩本次模型输入时，仍然可以保留完整执行历史用于审计和恢复。

## Session 保存的是引用顺序

Canonical Item（权威运行条目）是产品已经记录的用户输入、模型输出、工具调用或工具结果。它有稳定 ID。SDK 使用这些条目时，工程保存的是一份按顺序排列的 `itemRefs`，而不是再次复制一套聊天内容。[^session]

`PgSdkSession` 实现 SDK 的 `getItems`、`addItems`、`popItem` 和 `clearSession` 接口。底层视图按 Run 保存，`getSessionId()` 对 SDK 返回 Thread ID。这里可以看到外部会话标识与内部存储边界的区别：相同 Thread 的多次运行能够继承历史，各次执行仍有自己的持久引用视图。

读取历史时，代码从存储得到条目并按 SDK 的 `ModelItem` Schema 解析。追加时，它把 SDK 返回的对象连接到已经记录的事实 ID，再追加引用。

```ts
// 摘编自 PgSdkSession.getItems()。
const entries = await storage.readSdkSessionEntriesForExecution(
  principal,
  runId,
);

const items = entries.map(entry => {
  const item = protocol.ModelItem.parse(entry.canonical);
  this.known.set(item, {
    itemId: entry.itemId,
    digest: contentDigest(item),
  });
  return item;
});
```

这个对象到身份的映射有实际用途。SDK 可能弹出一个条目后重新追加它，程序需要认出这是原来的事实。另一方面，如果调用方修改了读出的条目，再请求按原身份追加，内容摘要检查会拒绝这种重定义。

正文相同也不等于条目相同。顾问两次输入“继续”，或模型两次输出相同文字，都是不同的发生时刻。源码使用对象引用、发生顺序和已记录事实的匹配关系保存身份；内容摘要只辅助匹配，不把文字当成全局唯一键。

## 输入、失败尝试和新回合怎样排列

一个新 Turn（用户回合）开始时，当前用户输入已经由请求受理事务保存。SDK 接收本次输入并持久化历史时，Session 必须复用这份 `userInputRef`，否则同一问题会出现两次。

工程的 `addItems()` 会检查 SDK 追加的用户文本是否匹配当前已受理输入，然后关联原输入 ID。ModelStep（模型调用步骤）和 ToolInvocation（工具执行实例）产生的条目，则应在各自边界先被记录，再由 Session 关联。[^session]

对于同一 Turn 的重试，失败 Run 的审计事实仍然保存。后续模型可见历史依据已接受的运行和当前尝试构造，避免把旧失败尝试的半成品自然混入新一轮输入。

`popItem()` 和 `clearSession()` 也有具体语义：它们修改 SDK 视图里的引用顺序和成员，产品权威历史继续保留。因此，“清空模型历史”与“删除业务会话”是不同操作。后者还有归属、运行状态和数据保留规则。

## 一次补查的输入到底增加了什么

下面是合成轨迹的内容摘要，用于观察两个模型请求的关系。它不是供应商原始报文。

```text
请求 1
  instructions：使用授权工具，根据证据说明条款
  input：用户问题
  tools：search_documents、get_source_page 等

模型输出 1
  callId=c1
  search_documents(query="示例保障甲 等待期", topK=5)

工具结果 1
  data：等待期定义片段
  sourceRefs：[已签发的来源引用]
  coverage：partial
  missing：与实际结果对应的元数据缺口

请求 2
  instructions：同一运行的指令
  input：用户问题 + 调用 c1 + c1 的工具结果
  tools：本次运行冻结且仍可执行的工具集合
```

模型读取请求 2 后，可能访问来源页，也可能改写查询。源码没有要求每一道题按同一套固定顺序调用全部工具。Harness 提供合法动作空间、持久反馈与资源限制，模型在其中选择下一步。

工程的真实自主执行证据曾记录检索 `topK` 从 5 调整到 10、读取来源和同 Thread 追问等行为。它们可以证明这类动作发生过；一次轨迹是否充分回答问题，还要结合实际来源和评测标准判断。本文采用合成结构解释输入变化，不复述受限业务原文。[^autonomy]

## 模型输入预算应作用于视图

随着工具调用增加，完整历史可能超过本次请求预算。工程使用 `projectModelRequest()` 生成新的输入数组，在 `RecordedModel` 调用具体 Provider 之前进行裁剪。[^budget]

裁剪单位是可以可靠匹配的完整历史 Turn。实现保留较新的连续历史后缀，同时保护当前 Turn、当前工具调用和结果，以及无法安全匹配的输入。工具调用与结果作为同一历史回合的事实处理，避免只保留一半协议。

```ts
// 摘编自 PgSdkSession.projectModelRequest()。
return {
  ...request,
  input: projectHistoricalTurns(
    request.input,
    this.history,
    currentTurnId,
    contextBudgetFor(this.environment.snapshot.modelConfig),
    this.completeTurnIds,
  ),
};
```

这里产生新的 `ModelRequest` 视图。Canonical 条目、Session 的完整引用、检查点和历史导出保持原有事实身份。否则，一次为了节省模型上下文的优化，可能无意中抹掉恢复所需的数据。

当前上下文预算使用 UTF-8 JSON 字节作为可确定的限制单位，例如历史字节、工具结果证据字节和整个请求字节。它们与模型的 token 上限不是同一个量。Token（模型分词单位）和字节之间没有固定换算比例，因此输出 token 设置与序列化字节预算要各自说明。

如果保护后的当前输入本身仍然超限，程序返回明确预算错误，而不是把一个工具结果从中间截成不完整 JSON。业务层可以减少证据体积、改变返回结构，或在后续允许的运行中调整策略。

## 为什么恢复也要理解 Session

Checkpoint（检查点）保存的不只是一个“已经执行到第几步”的数字。它还需要知道当时哪些条目已经提交，模型历史引用的顺序是什么，工具调用和结果是否成对，以及恢复时使用的配置是否兼容。

工程曾针对已有 Session 被 `pop` 或 `clear` 后的恢复边界建立回归。若检查点重新从另一套默认历史取数据，恢复的模型输入就会与中断前的视图不同。实现因此让检查点引用使用与 Session 初始化一致的选择规则。[^recovery]

这也是持久引用视图的价值：既可以保留完整审计事实，也可以准确描述本次执行真正使用的历史。

## 怎样验证模型看到了正确的内容

最直接的办法，是让受控模型检查自己收到的请求。第一次返回固定工具调用，第二次断言输入中包含对应 `callId` 的结果，再给出最终内容。需要观察的不仅是结果文字，还包括输入是否只出现一次、旧失败条目是否混入、调用结果是否成对。

另一个测试可以使用两个文本完全相同的用户回合。系统应产生两个不同的输入 ID，并按回合顺序读取。对上下文裁剪，则构造几个完整旧回合和一个当前工具结果，检查旧回合以完整单位移出视图，而当前协议保持完整。

这些测试把“Agent 记住了什么”转化成可检查的数据关系。真实模型是否善于利用这些上下文，是另一层行为评测。

## 连续性来自事实和视图的分工

Agent 的继续查询不是一段神秘的长时间思考。它由多次模型请求组成，每次请求都依赖已经取得的证据。Harness 的职责是让这些证据准确进入后续输入，同时保留足够的事实用于复核和恢复。

权威条目保存发生过什么，Session 保存本次执行使用的历史顺序，ModelRequest 选择本次调用可承载的内容。三层分开，系统才能同时拥有连续性、可追溯性和预算控制。

延伸阅读：[预算、取消与检查点恢复](06-budgets-cancellation-recovery.md)。

## 实现依据

[^session]: Insurance Agent Runtime 源码快照 `c66112dfd8676b47b143dc2e26739278cf4b0dde`：`apps/runtime/src/runner/session.ts`，`PgSdkSession.create/getItems/addItems/popItem/clearSession`；以及 `storage/store.ts` 的 SDK Session 引用视图方法。
[^tool]: 同一快照：`packages/contracts/src/tools.ts` 的实际工具结果字段。本文中“证据要件”是业务分析维度，不代表代码中另有固定规划器。
[^budget]: 同一快照：`apps/runtime/src/runner/context-budget.ts`、`runner/model.ts` 和 `docs/implementation/context-budget-preparation.md`。
[^recovery]: 同一快照中的 `docs/implementation/recovery-boundaries-fix.md`：Session 引用选择、恢复边界及相关数据库回归记录。
[^autonomy]: `docs/implementation/evidence/M13-original-dod-gate-20261008.json` 保存的历史自主行为证据；该证据绑定其自己的执行源码，不能解释为当前文章新执行的实验或普遍质量结论。
[^context]: [OpenAI Agents SDK：Context](https://openai.github.io/openai-agents-js/guides/context/) 与 [Sessions](https://openai.github.io/openai-agents-js/guides/sessions/)。用于理解 SDK 的接口职责，项目具体持久化方式见上述源码。
