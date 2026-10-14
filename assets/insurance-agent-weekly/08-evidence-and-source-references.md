# 从检索片段到原始条款：证据身份、来源引用与多跳查询

“哪些产品支持某项保单功能？”和“找几段介绍这项功能的文字”，表面上很相似，执行要求却不同。前者需要确定产品范围，逐项检查，再说明哪些对象有支持证据；后者只需要返回一组相关片段。

当 Agent 开始跨产品、跨文档查询，结果会经过多个工具。产品目录提供标识，检索服务提供片段，来源工具读取原文页，模型再整合结论。每次传递都需要保持身份，否则一段正确文字也可能被关联到错误产品或错误版本。

本文讨论 Harness（智能体执行控制系统）怎样保存这条证据链。工程实例为 Insurance Agent Runtime，业务例子采用合成产品，不包含真实条款正文。

## 多跳查询先需要一个明确范围

Multi-hop Retrieval（多跳检索）表示后一次查询依赖前一次取得的信息。例如，先根据别名找到产品正式身份，再以该产品查询免责条款，最后沿检索结果中的来源引用读取整页。

```text
产品别名
  → 产品目录：正式产品 ID
  → 检索：与该产品匹配的条款片段
  → 来源读取：片段所属文档的指定页
  → 模型回答：结论与来源关联
```

这条路径可以由模型自主选择。Harness 不需要额外生成一份写死的“查目录、再检索、再扩页”业务脚本，但每个动作都必须携带可核对的输入和返回结果。

对于枚举任务，还要先确认搜索空间。目录返回 `nextCursor`（下一页游标），表示仍有后续数据；检索的 `topK` 只限制本次最多返回多少个片段。两者不能互相替代：取到了五段相关文字，不等于已经检查了全部产品。

可以把业务要求表示为“候选对象集合中，每个对象目前取得了哪些证据”。这是一种任务分析视图，并不代表当前源码中存在独立的固定规划器。实际运行仍由模型选择工具，工具结果和执行轨迹提供观察依据。

## 五种标识各自回答不同问题

检索结果和原文读取涉及多个 ID。给它们分清职责，比把所有字段统一叫 `id` 更重要。

| 标识 | 含义 | 典型用途 |
| --- | --- | --- |
| 产品 ID | 业务目录中的产品身份 | 过滤和实体匹配 |
| `chunkId` | 检索索引中的片段身份 | 定位命中片段 |
| `documentId` | 源文档身份 | 文档关联和评测定位 |
| `evidenceId` | 返回证据条目的身份 | 记录某次获得的证据 |
| `sourceRef` | 服务端签发的授权来源引用 | 在当前权限下读取指定来源 |

原始上传 ID、机构内部文档 ID、文件内容摘要也可能存在。它们应保留自己的来源和语义。尤其是向量检索返回的文档条目 ID，在某些服务里实际上是 chunk ID，不能直接当作原始 PDF 的身份。

工程的 `search_documents_output` 显式保留了这些差别：证据行包含片段 ID、可空的文档 ID、可空的页码、正文、产品与保司信息、检索分数及其度量方式。来源尚未充分确认时，`sourceRef` 可以为空，`metadataStatus` 则报告元数据完整程度。[^contracts]

```json
{
  "evidenceId": "SYN-E-01",
  "sourceRef": null,
  "chunkId": "SYN-CHUNK-27",
  "documentId": null,
  "fileName": "示例条款.pdf",
  "page": 12,
  "text": "合成片段：此处包含待核验的适用条件。",
  "company": "示例保司",
  "productName": "示例保障甲",
  "score": 0.76,
  "scoreMetric": "synthetic_score",
  "chunkType": "text",
  "metadataStatus": "partial",
  "provenance": {
    "mode": "synthetic",
    "backendId": "example-retrieval",
    "datasetVersion": "example-v1",
    "observedAt": "2026-01-01T00:00:00Z"
  }
}
```

这是合成的单条证据行，外层结果还包含状态、引用集合和缺失说明。片段已经返回，但文档身份尚未确定，所以不能据此发行一个声称完整有效的来源引用。

实际 ToolIO（工具输入输出协议）中，搜索顺序由结果数组表达。实验导出器可以据此计算 rank（排名），而不必假设上游已经返回了某个名为 rank 的字段。

## SourceRef 是服务端管理的引用

Opaque Reference（不透明引用）意味着调用方按原值传递，由服务端解释。模型不需要知道存储对象路径，也不负责拼接下载地址。

工程定义 `SourceReferenceDirectory`（来源引用目录），提供发行和解析接口：[^directory]

```ts
export interface SourceReferenceDirectory {
  readonly persistence: 'process' | 'durable';
  issue(
    locator: SourceLocatorCandidate,
    context: BusinessContext,
  ): Promise<string>;
  resolve(
    sourceRef: string,
    context: BusinessContext,
  ): Promise<ControlledSourceLocator | null>;
}
```

Locator（来源定位信息）保存文档对象、页码、版本、机构和主体等内容。合成测试可以使用进程内目录，真实模式则要求持久目录，保证进程重启后仍能解析已签发引用。

`PgSourceReferenceDirectory` 将这些关系保存到 PostgreSQL。引用由包含命名空间、数据模式、主体、对象、页码和版本等字段的固定元组计算摘要。相同元组能够得到稳定引用，但摘要本身不是权限。解析时仍然必须匹配当前主体、有效期和文档授权。[^pg]

这点很重要：一个人即使取得了另一个主体的来源字符串，也不能因此拥有读取文档的权利。引用是查找入口，授权由服务端策略决定。

## 读取来源的过程包含两种授权

SourcePort（来源读取接口）需要同时处理资源授权和文档授权。

资源授权决定当前调用能否访问某个上游对象服务；文档授权决定当前主体能否读取具体文档版本和页面。二者可以来自不同的规则，因此源码把它们分开检查。

`createParsedJsonSourcePort()` 的主要步骤是：解析来源引用，验证定位信息和后端版本，读取已经授权的解析文档，选择指定页，生成标准输出，再检查引用和授权条件仍然符合要求。[^source]

```ts
// 摘编自 createParsedJsonSourcePort()。
const found = await directory.resolve(input.sourceRef, context);
if (!found) throw new RegistryError('TOOL_SCOPE_DENIED');

const locator = assertSourceLocator(
  found,
  context.principal,
  input.sourceRef,
);

const document = parseParsedSourceDocument(
  await client.requestJson(
    'GET_PARSED_OBJECT',
    '/' + locator.objectKey,
    context,
  ),
  {
    landingAiPageBase: locator.landingAiPageBase,
    pageMappingVerified: locator.pageMappingVerified,
  },
);

const blocks = selectParsedSourcePage(document, locator.defaultPage);
```

完整实现还包含缺页状态、缓存、页码验证、信号检查和返回前复核。一个 `sourceRef` 在当前模型里选择一页，页内多个解析块再按顺序组合成返回内容。

检查发生在异步调用的前后，因为查询授权或读取对象本身也需要时间。但这组检查仍然是有界的当前状态观察，不是横跨所有上游服务的分布式事务。文章中的“再次核验”应按这个实际保证理解。

## 页码转换必须绑定实际解析格式

PDF 物理页、正文印刷页码、解析数组下标是三个不同概念。一个 PDF 的数组第 0 项通常对应物理第 1 页，但具体解析格式可能将页码单独存储，甚至采用不同起点。

因此，来源定位信息需要明确解析器的页码基准，并用原始 PDF 核验。对文字页、表格页和中部页面分别验证，能发现只检查第一页时容易遗漏的偏移。

工程的来源组件支持解析格式归一，输出 `pageMappingVerified`（页码映射已核验）。已保存的组件证据曾针对两组原始 PDF 与解析文档进行首部、中部和表格页读取。这个结果支持的是已核验的具体样本与映射，不是全量文档页码天然正确。[^pages]

解析内容还可能混合正文和表格。当前 SourcePort 将页内文本顺序连接，在格式混合时保留警告，并用实际输出说明。这不等同于原始 PDF 的视觉重建。涉及复杂表格的业务判断，仍需核对表头、单位、跨页延续和注释。

## 缓存需要保存数据来源的时间

反复读取同一来源页，缓存能够减少网络请求和解析开销。但缓存命中并不意味着授权仍然有效。

源码先执行当前文档与资源检查，再查询缓存；下载后准备写入缓存时也进行复核。缓存按受控资源、主体和来源定位信息使用，防止不同版本或不同归属混用。[^source]

另一个细节是时间。读取缓存并不会使旧数据变成刚从上游取得的新数据。业务结果中的 `observedAt`（观察时间）和版本需要保持真实语义，后续模型或报告才能判断证据是否足够新。

来源引用也有 TTL（有效期）。活动引用的普通发行不延长原有效期；过期后，只有当前授权仍然成立才允许重新发行或续期。解析操作本身不会静默续命。即使授权查询耗时较久，解析结束前仍需确认引用没有在等待期间过期。[^pg]

这让“原来读得到”和“现在仍可读取”成为两种可以解释的状态。

## 多跳结果怎样连接回执行历史

每次工具结果首先保存为 Canonical Item（权威运行条目）。ToolInvocation（工具执行实例）引用调用条目和结果条目，结果中再关联 `sourceRef`。模型下一次请求通过 Session（会话历史适配器）取得这些已提交事实。

```text
Run
  → ModelStep：模型选择查询
  → ToolInvocation：执行查询
  → resultItemRef：已保存工具结果
  → evidenceId / sourceRef：业务证据与来源
  → ModelStep：模型读取结果并继续
  → finalContentRef：最终已提交内容
```

最终答案可能引用多份工具结果。系统能够通过这条关系定位其输入来源，但“引用的文字确实支持结论”还需要语义评审。数据库中存在引用，证明关联可解析；人工核验或校准后的评审，证明业务断言得到支持。

对于全量枚举任务，还要检查过程覆盖。目录分页是否完整，哪些对象从未查询，哪些字段是未知值，都可能影响最终结论。一个 `coverage=complete` 必须按具体工具协议理解其范围，不能直接扩大为整个市场或所有产品均已验证。

## 三个能暴露真实问题的验证案例

第一个案例是来源身份不完整：检索正文可返回，但缺失原始文档身份。测试应确认元数据缺口被保留，来源引用按准入规则处理，模型没有凭一个文件名自行构造来源。

第二个案例是来源权限变化：先签发引用，再改变测试授权。再次读取时，应在返回正文前得到相应拒绝或不可用状态。缓存命中同样经过这个流程。

第三个案例是跨步骤引用：先完成检索，保存结果，再由另一工具读取那个精确 `sourceRef`。检查第二次调用使用的引用来自已授权结果，页码和版本匹配，最终历史能够回溯两次调用。

这些测试把多跳查询的可靠性落实到可验证连接上。查询策略可能多样，身份、权限和结果提交则遵循明确规则。

## 证据链让复杂答案有可追溯的基础

复杂查询的价值不只是多调用几个工具，而是从一个信息缺口走到可以核验的证据。稳定的对象身份、规范化结果和授权来源目录，让每一次跳转都有明确含义。

在这条基础上，模型才有可能把目录、检索和原文读取组合成有效行动。下一篇从动作空间本身出发，解释工具目录如何定义、选择和冻结：[Tool Registry 与运行快照](09-tool-registry-and-snapshots.md)。

## 实现依据

[^contracts]: Insurance Agent Runtime 源码快照 `c66112dfd8676b47b143dc2e26739278cf4b0dde`：`packages/contracts/src/tools.ts` 的搜索、目录和来源结果 Schema。示例 JSON 是合成证据行。
[^directory]: 同一快照：`apps/runtime/src/business/adapters/source-references.ts`，`SourceReferenceDirectory` 与合成目录接口。
[^pg]: 同一快照：`apps/runtime/src/business/adapters/pg-source-references.ts`，`PgSourceReferenceDirectory.issue`、`resolve` 和元组身份校验。
[^source]: 同一快照：`apps/runtime/src/business/adapters/source.ts`、`parsed-source.ts`、`source-cache.ts`。正文分析来源读取与当前授权边界。
[^pages]: 工程历史证据 `docs/implementation/evidence/M09-real-component-gate-20261007.json`、`M10-real-component-gate-20261007.json`。来源样本验证保留其原执行版本，本文不将其扩展为全量数据质量结论。
