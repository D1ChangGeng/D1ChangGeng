# 部署生命周期、数据库恢复与发布证据｜内容大纲

从把已通过测试的 Harness 交给另一台机器运行开始。用户需要知道当前部署是什么版本、数据放在哪里、怎样安全停机，以及升级或数据库故障后怎样恢复相同的运行事实。

对应正文：[把 Harness 交付出去：部署生命周期、数据库恢复与版本证据](../13-deployment-and-release.md)。

## 01｜代码能运行以后，交付还包含哪些对象？

区分源码、构建输入、镜像、实例配置、数据库、模型与业务资源。用一个合成部署示例建立完整物理形态。

## 02｜一个本地 Runtime 部署实际长什么样？

解释 Node 容器、PostgreSQL 容器、loopback HTTP、持久卷与私有配置。说明 Linux/WSL 已记录路径和其他平台需独立验证。

## 03｜怎样把构建产物绑定到准确源码？

展示 Git commit/tree、工作树修改、git archive 和镜像 revision label。区分标签、镜像ID和RepoDigest，不把标签命名当成内容不可变保证。

## 04｜Dockerfile 怎样固定运行环境和依赖？

读取 Node镜像摘要、pnpm版本、frozen lockfile和两阶段构建。说明实际镜像保留的依赖范围及构建检查的具体覆盖。

## 05｜为什么构建上下文需要允许列表？

说明源码与私有配置、日志、dump和本地依赖隔离。列出可核对文件边界，保留具体检查方法。

## 06｜实例 Manifest 与私有配置是什么？

展示 project、database、activeDatabase、端口、image、UID/GID和配置路径的形状。区分非秘密身份和秘密值，解释独立实例命名。

## 07｜环境变量与 Compose 如何避免覆盖实例身份？

复盘实际 manifest-derived environment、显式env-file和rendered config检查。解释 caller directory绑定及串行运维，避免旧ops.env说明被误用为当前权威。

## 08｜怎样从干净源码启动一个隔离实例？

提供实际git archive、docker build、local.mjs init/up命令，使用新目录和本地合成模式。明确操作者需要已有授权Docker环境。

## 09｜Readiness 与能力目录分别证明什么？

检查数据库/队列就绪、工具模式和可用状态，以及实际Run的终态。健康接口不代表真实模型或真实业务已完成验证。

## 10｜运行中的数据到底保存在哪里？

区分Runtime临时内存、PostgreSQL volume、配置目录和外部业务资料。说明down保留卷、clearSession不删除历史等相关语义。

## 11｜停机时为什么先结束执行再关闭存储？

结合composeRuntime dispose、Worker stop、SSE清理和租约监督。区分正常停止与强制进程退出。

## 12｜数据库迁移怎样验证历史语义？

说明Drizzle迁移、journal、锁、重复执行与最终引用读回。引用历史4→5→5测试但绑定原记录范围，不将其当作本轮执行。

## 13｜一份数据库备份包含什么、缺少什么？

说明pg_dump custom格式、状态/事件/检查点/Job，以及凭证、集群角色和外部资料的独立备份。介绍恢复权限与数据保留范围。

## 14｜怎样在新数据库验证恢复？

沿stop、backup、restore到新库、读回、activate、up说明。展示dump摘要和参数化查询，避免覆盖已有数据库。

## 15｜活动数据库切换为什么不是简单改一个值？

分析container/host配置与manifest.activeDatabase的一致性、多文件rename非整体事务、中断拒绝与恢复配置。后续backup应使用activeDatabase。

## 16｜镜像回滚和检查点恢复有什么不同？

区分代码回滚、数据恢复和Run Resume，说明SDK、模型、工具、Schema兼容条件。兼容旧镜像证据不能扩大为任意历史downgrade保证。

## 17｜怎样验证部署隔离和资源安全？

读取非root、只读根文件系统、cap_drop、no-new-privileges、受控挂载与CPU/内存限制。结合config tests和实际容器读回。

## 18｜一个发布记录怎样连接源码、测试、镜像和数据？

给出版本证据矩阵和可运行的快照一致性比较示例。区分静态检查、历史镜像生命周期和真实业务验证。

## 19｜怎样运行现有运维测试并判断交付完成？

说明config-only测试与Docker/PG生命周期脚本的不同效果，已有端口/共享服务保持不变等范围。最后连接端到端实践篇。

## 实现与材料

源码基准 `c66112dfd8676b47b143dc2e26739278cf4b0dde`。主要依据 Dockerfile、`tools/ops/local.mjs`、`composition/runtime.ts`、`docs/implementation/local-ops-preparation.md` 及其绑定的历史证据。文中使用公开相对路径和合成身份，不发布私有实例目录或凭证。

正文包含部署关系图、构建身份链、启动/停机时序、新库恢复图与兼容矩阵。HTML折叠区域提供配置摘要、风险边界和读回示例。所有操作命令仅作为授权隔离环境的实践入口，本轮不执行真实部署或数据库变更。
