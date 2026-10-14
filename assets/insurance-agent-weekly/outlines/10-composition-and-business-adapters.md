# 依赖装配、业务接口与真实后端接入｜内容大纲

从一个具体接入任务开始：搜索工具已在合成环境跑通，接下来要连接检索网关、产品目录和原文读取服务。围绕“工具名称保持稳定，后端实现逐步替换”解释 Port、Adapter、Binding 和 Composition Root。

对应正文：[把工具接到真实业务系统：依赖装配、Port、Adapter 与数据模式](../10-composition-and-business-adapters.md)。

## 01｜合成工具能运行以后，真实接入还要完成什么？

用字段、权限、分页、单位和版本说明网络连接成功与业务能力可用的差别。保留检索、目录、来源三个贯穿接口。

## 02｜Port 是什么？一个业务接口实际长什么样？

展示 RetrievalPort、SourcePort、CatalogPort 与 BusinessInput/Output。解释 TypeScript interface 的作用和运行时校验边界。

## 03｜BusinessContext 与模型参数怎样分开？

展示 principal、signal、runId、snapshotId、backendRevision、dataMode。说明 content 字段移除、可信主体注入与服务端依赖。

## 04｜Adapter 怎样把两种 API 转为一个工具能力？

用两份合成检索回包说明参数与结果转换、错误类型与来源字段。提供可运行的纯转换示例，并区分结构转换和实际网络访问。

## 05｜BackendBinding 怎样捕获实际执行方法？

读取 bindBusinessPorts，展示能力分派、方法绑定、描述校验和集合版本。说明凭证与客户端在闭包中，而非进入 Snapshot。

## 06｜Composition Root 怎样把组件装配成服务？

展示 composeRuntime 与业务装配工厂，说明数据库、Registry、HTTP、Worker 和模型依赖的建立顺序。解释本地依赖注入与 SDK 参数的关系。

## 07｜Synthetic、Recorded、Real 各代表什么？

将业务数据模式与模型执行模式作为两条轴。介绍固定合成、请求匹配回放和真实资源，说明它们各自适合的测试。

## 08｜怎样设计保留业务难点的合成夹具？

覆盖空结果、partial、未知字段、无页码、过期优惠和分页。保持单位、有效期、身份与来源语义。

## 09｜真实业务配置和授权回执怎样组织？

阅读 BusinessComposition、ResourceGrant 和 Readback Schema。展示脱敏字段结构，解释配置摘要、当前权限与独立资源核验凭据的关系。

## 10｜为什么私有配置解析不等于真实读回？

明确 privateBusinessReadbackVerifier 只消费独立验证证据，不产生真实接口验证。解释没有 verifier 时资源保持待准入。

## 11｜Scoped HTTP Client 怎样限制实际请求？

根据 HTTP 客户端与配置源码说明资源、操作、方法、路径、认证、重定向、返回体和取消。对未读取行为不做额外保证。

## 12｜目录和检索之间如何保持实体与分页语义？

分析 productId 到上游名称过滤的解析、游标、最大解析页数和缺失实体。明确不能只把第一页结果当成全目录。

## 13｜来源读取和结构化事实怎样接入？

介绍持久 SourceReferenceDirectory、attachPool、文档页授权，以及工程事实数据集的独立准入。区分真实业务数据、测试夹具和实时接口。

## 14｜懒初始化与多阶段请求怎样共享授权和期限？

说明动作前后当前检查、信号传播、单项能力总期限与内部请求时间。使用配置版本中的 operational timeout alignment 说明实际取舍。

## 15｜怎样沿真实 Adapter 分支做低成本联调？

先用本地合成 HTTP 响应覆盖真实解析，再执行授权接口读回，最后接入真实模型。每一层保留自己的测试结论。

## 16｜启动失败和服务关闭怎样释放依赖？

介绍数据库附加、Worker/HTTP 关闭、客户端和缓存的生命周期。明确图上依赖需要有实际构造和释放函数。

## 17｜如何从源码和运行结果判断接入完成？

提供 Port 契约、Descriptor、Snapshot、工具结果、Invocation 与事件的核对清单，串联一个真实单工具路径。

## 18｜稳定能力怎样支持后端持续演进？

收束工具名称、业务契约、Adapter、绑定和部署版本的关系，连接持久事件与流式输出专题。

## 依据与展示

实现基准为 `c66112dfd8676b47b143dc2e26739278cf4b0dde`。重点文件：`business/ports.ts`、`registry/backends.ts`、`business/composition/configured-business.ts`、`business/adapters/http.ts`、目录与检索 Adapter、`composition/runtime.ts`。

正文使用接口类型、转换函数、依赖装配图、数据模式矩阵和联调时序。HTML 折叠区展示较长配置摘录与边界测试。示例均说明是纯函数、代码摘编或待连接受控实例的操作；公开正文保留必要版本依据。
