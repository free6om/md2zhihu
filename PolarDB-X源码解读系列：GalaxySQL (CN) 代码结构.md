本文主要介绍 GalaxySQL 代码结构，首先简要回顾 PolarDB-X 的架构，然后从目录入手介绍各个模块的功能，最后列出一些关键接口便于读者调试代码。

# 整体架构

![Architecture](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/4239/1636350574242-f2ecdbfe-9bf1-442f-97ea-72649987534e.png)

PolarDB-X 包含 4 个核心组件构成，CN (Compute Node) 负责计算、DN (Data Node) 负责存储、GMS (Global Meta Service) 负责管理元数据和提供TSO服务，CDC (Change Data Capture) 负责生成变更日志。其中，CN 作为服务入口，完成三个任务：

1.  通过 MySQL 协议接收用户请求，返回结果
1.  作为分布式查询引擎，兼容 MySQL 语法，提供分布式事务、全局索引、MPP 等特性
1.  通过 RPC 协议与 DN 交互，下发读写指令，汇总结果

下文将结合 CN 的三个任务，介绍代码、目录、模块在其中扮演的角色，并给出一些关键接口，便于读者继续探索。

# 代码简介

CN 代码托管在 GitHub 上，划分为 [GalaxySQL](https://github.com/ApsaraDB/galaxysql) 和 [GalaxyGlue](https://github.com/ApsaraDB/galaxyglue) 两个仓库。MySQL 协议实现和分布式查询引擎包含在 GalaxySQL 仓库中，由于 License 原因，与 DN 交互的 RPC 协议相关代码单独放在 GalaxyGlue 中。

> 调试代码前，首先需要下载两个仓库的代码。推荐首先下载 GalaxySQL 代码，然后通过 git submodule 引入 GalaxyGlue，这样可以独立提交两个仓库的变更，具体步骤参考 [contributing 文档](https://github.com/ApsaraDB/galaxysql#contributing)


CN 是一个多模块的 Java 项目，模块之间通过接口暴露服务，模块关系记录在 [pom.xml](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/pom.xml#L159) 中，通过 `mvn dependency:tree` 命令可以查看全部依赖。

> 部分接口使用了 [SPI](https://docs.oracle.com/javase/tutorial/sound/SPI-intro.html) 机制，这部分接口需要在模块的 `src/main/resources/META-INF/polardbx` 目录下查看当前模块中使用的具体实现。


关于编译打包，可以参考 [编译/初始化文档](https://github.com/ApsaraDB/galaxysql/blob/main/docs/en/quickstart-development.md#compiling-polardb-x-compute-node-galaxysqlgalaxyglue)。CN 的 main 方法在 `polardbx-server` 模块下的 [TddlLauncher](https://github.com/ApsaraDB/galaxysql/blob/0d7b1c5507c6b9c33c76e4e023983eb0278d52bf/polardbx-server/src/main/java/com/alibaba/polardbx/server/TddlLauncher.java#L62) 类中，"Tddl" 代表 "Taobao Distributed Data Layer"，产生于 [PolarDB-X 0.5 时代](https://zhuanlan.zhihu.com/p/289870241)，由于类名被关联系统依赖，保留至今。

# 目录与模块

[GalaxySQL](https://github.com/ApsaraDB/galaxysql) 项目首页中可以看到很多目录和文件，加上重名为 polardbx-rpc 后加入工程的 [GalaxyGlue](https://github.com/ApsaraDB/galaxyglue) ，项目根目录下包含 13 个文件夹，9 个文件。其中，代码目录以 polardbx 开头，docke_build.sh 用于本地生成 docker 镜像，pom.xml 是整个项目的工程文件，saveVersion.sh 用于打 RPM 包时生成版本号后缀。

![directory-and-files](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/4239/1636363375184-0b6fbbf5-08ab-42ae-b2bc-4d6e0f04009d.png)

根目录中，每个以 polardbx 开头的目录，都代表一个独立的模块，以下展开介绍各个模块

<table>
<colgroup>
<col style="width: 33%" />
<col style="width: 33%" />
<col style="width: 33%" />
</colgroup>
<tr class="header">
<th>模块</th>
<th>包</th>
<th>简介</th>
</tr>
<tr class="odd">
<td>polardbx-calcite</td>
<td>–</td>
<td>PolarDB-X 优化器基于 <a href="https://calcite.apache.org/">Apache Calcite</a> 框架做了深度定制，该模块用于引入 calcite-core 的代码，定制代码在 polardbx-optimizer 模块下</td>
</tr>
<tr class="even">
<td>polardbx-calcite</td>
<td>org.apache.calcite.plan.hep</td>
<td>RBO 框架</td>
</tr>
<tr class="odd">
<td>polardbx-calcite</td>
<td>org.apache.calcite.plan.volcano</td>
<td>CBO 框架</td>
</tr>
<tr class="even">
<td>polardbx-calcite</td>
<td>org.apache.calcite.sql</td>
<td>AST (抽象语法树) 节点数据结构</td>
</tr>
<tr class="odd">
<td>polardbx-calcite</td>
<td>org.apache.calcite.rel</td>
<td>逻辑计划相关代码，包括数据结构、优化规则、类型系统以及相关工具类等内容，主要包含 calcite-core 的原始代码，增加的逻辑/物理算子和优化规则放在 polardbx-optimizer 模块下</td>
</tr>
<tr class="even">
<td>polardbx-calcite</td>
<td>org.apache.calcite.rex</td>
<td>表达式相关代码，主要包含 calcite-core 的原始代码，新增的表达式相关代码放在 polardbx-optimizer 模块下</td>
</tr>
<tr class="odd">
<td>polardbx-calcite</td>
<td>org.apache.calcite.schema</td>
<td>Catalog 相关接口抽象，具体实现在 polardbx-optimizer 模块下</td>
</tr>
<tr class="even">
<td>polardbx-common</td>
<td>–</td>
<td>工具类模块</td>
</tr>
<tr class="odd">
<td>polardbx-executor</td>
<td>–</td>
<td>执行器模块，所有执行器代码都在这里，比如 SQL 执行代码，MPP 执行器，异步 DDL 引擎，统计信息收集，scaleout，全局二级索引等，内容比较多，后续文章中展开介绍</td>
</tr>
<tr class="even">
<td>polardbx-gms</td>
<td>–</td>
<td>元数据服务模块</td>
</tr>
<tr class="odd">
<td>polardbx-gms</td>
<td>com.alibaba.polardbx.gms.listener</td>
<td>后台监听 metaDb 配置的接口，用于支持 CN 感知配置变化</td>
</tr>
<tr class="even">
<td>polardbx-gms</td>
<td>com.alibaba.polardbx.gms.metadb</td>
<td>读写 schema, table, column, index 等元数据的接口，informaction_schema 中展示的信息大部分通过这里的接口获取。metaDb 是 PolarDB-X 持久化元数据的系统库</td>
</tr>
<tr class="odd">
<td>polardbx-gms</td>
<td>com.alibaba.polardbx.gms.privilege</td>
<td>授权、鉴权、白名单、读写权限信息相关接口</td>
</tr>
<tr class="even">
<td>polardbx-gms</td>
<td>com.alibaba.polardbx.gms.sync</td>
<td>CN 节点间通信相关接口</td>
</tr>
<tr class="odd">
<td>polardbx-gms</td>
<td>com.alibaba.polardbx.gms.topology/ha/locality/partition/tablegroup</td>
<td>数据拓扑相关接口，记录了物理分区分布(topology)，leader/follower(ha)，地域(locality)，分区(partition)，表组(tablegroup) 等信息</td>
</tr>
<tr class="even">
<td>polardbx-net</td>
<td>–</td>
<td>MySQL 客户端/服务器协议相关接口</td>
</tr>
<tr class="odd">
<td>polardbx-net</td>
<td>com.alibaba.polardbx.net</td>
<td>包含TCP包处理类和与客户端交互的协议接口（包括登陆鉴权、文本协议、Prepared Statement等）</td>
</tr>
<tr class="even">
<td>polardbx-net</td>
<td>com.alibaba.polardbx.rpc.cdc</td>
<td>CDC 组件变更日志传递相关接口</td>
</tr>
<tr class="odd">
<td>polardbx-net</td>
<td>com.alibaba.polardbx.ssl</td>
<td>SSL 相关处理接口</td>
</tr>
<tr class="even">
<td>polardbx-optimizer</td>
<td>–</td>
<td>优化器模块</td>
</tr>
<tr class="odd">
<td>polardbx-optimizer</td>
<td>com.alibaba.polardbx.optimizer.config.meta</td>
<td>包含从逻辑/物理计划中收集信息的工具类，比如统计查询引用了哪些列，计算查询的统计信息等</td>
</tr>
<tr class="even">
<td>polardbx-optimizer</td>
<td>com.alibaba.polardbx.optimizer.config.schema/table</td>
<td>库/表元数据相关实现，实现了 org.apache.calcite.schema 中定义的元数据接口</td>
</tr>
<tr class="odd">
<td>polardbx-optimizer</td>
<td>com.alibaba.polardbx.optimizer.core.datatype</td>
<td>数据类型相关代码</td>
</tr>
<tr class="even">
<td>polardbx-optimizer</td>
<td>com.alibaba.polardbx.optimizer.core.function</td>
<td>表达式计算相关代码</td>
</tr>
<tr class="odd">
<td>polardbx-optimizer</td>
<td>com.alibaba.polardbx.optimizer.core.planner</td>
<td>RBO/CBO 优化规则</td>
</tr>
<tr class="even">
<td>polardbx-optimizer</td>
<td>com.alibaba.polardbx.optimizer.core.profiler</td>
<td>执行过程中，收集运行时资源消耗信息</td>
</tr>
<tr class="odd">
<td>polardbx-optimizer</td>
<td>com.alibaba.polardbx.optimizer.core.rel</td>
<td>逻辑/物理算子实现，DQL/DML/DDL/DAL 语句经过优化器后，转化为物理算子构成的执行计划</td>
</tr>
<tr class="even">
<td>polardbx-optimizer</td>
<td>com.alibaba.polardbx.optimizer.core.hint</td>
<td>HINT 处理相关代码</td>
</tr>
<tr class="odd">
<td>polardbx-optimizer</td>
<td>com.alibaba.polardbx.optimizer.core.sequence</td>
<td>全局唯一 ID 相关代码</td>
</tr>
<tr class="even">
<td>polardbx-optimizer</td>
<td>com.alibaba.polardbx.optimizer.core.rule/sharding</td>
<td>分区裁剪相关代码</td>
</tr>
<tr class="odd">
<td>polardbx-optimizer</td>
<td>com.alibaba.polardbx.optimizer.core.view</td>
<td>内置视图的执行计划节点，主要为 information_schema 和 SHOW 指令涉及的视图</td>
</tr>
<tr class="even">
<td>polardbx-parser</td>
<td>–</td>
<td>语法解析模块，基于<a href="https://github.com/alibaba/druid">alibaba/druid</a> 内置的 parser</td>
</tr>
<tr class="odd">
<td>polardbx-rpc</td>
<td>–</td>
<td>与 DN 通信的相关代码</td>
</tr>
<tr class="even">
<td>polardbx-rpc</td>
<td>com.alibaba.polardbx.rpc</td>
<td>rpc 协议交互代码</td>
</tr>
<tr class="odd">
<td>polardbx-rpc</td>
<td>com.alibaba.polardbx.mysql.cj</td>
<td>rpc 协议数据对象</td>
</tr>
<tr class="even">
<td>polardbx-rule</td>
<td>–</td>
<td>拆分规则模块</td>
</tr>
<tr class="odd">
<td>polardbx-server</td>
<td>–</td>
<td>服务入口，将其他模块组织在一起，main 方法在该模块中</td>
</tr>
<tr class="even">
<td>polardbx-server</td>
<td>com.alibaba.polardbx.cdc</td>
<td>CDC 核心控制模块，包括 CDC 系统库表维护，CDC 元数据初始化，CDC DDL 打标等内容</td>
</tr>
<tr class="odd">
<td>polardbx-server</td>
<td>com.alibaba.polardbx.config</td>
<td>系统配置加载相关代码</td>
</tr>
<tr class="even">
<td>polardbx-server</td>
<td>com.alibaba.polardbx.manager</td>
<td>管理指令相关代码，用于处理从管理端口传入的指令</td>
</tr>
<tr class="odd">
<td>polardbx-server</td>
<td>com.alibaba.polardbx.matrix</td>
<td>SQL 处理内部入口，串联解析，优化，执行流程</td>
</tr>
<tr class="even">
<td>polardbx-server</td>
<td>com.alibaba.polardbx.server</td>
<td>包含 main 方法和请求处理主流程相关代码</td>
</tr>
<tr class="odd">
<td>polardbx-server</td>
<td>com.alibaba.polardbx.server.handler</td>
<td>解析文本协议和LoadData指令的相关代码</td>
</tr>
<tr class="even">
<td>polardbx-server</td>
<td>com.alibaba.polardbx.server.response</td>
<td>部分管控指令的执行代码</td>
</tr>
<tr class="odd">
<td>polardbx-transaction</td>
<td>–</td>
<td>事务管理模块，包含分布式事务的实现代码</td>
</tr>
<tr class="even">
<td>polardbx-transaction</td>
<td>com.alibaba.polardbx.transaction.async</td>
<td>异步任务相关代码，包括死锁检测、事务超时、事务日志滚动等</td>
</tr>
<tr class="odd">
<td>polardbx-transaction</td>
<td>com.alibaba.polardbx.transaction.log</td>
<td>事务日志相关代码</td>
</tr>
<tr class="even">
<td>polardbx-transaction</td>
<td>com.alibaba.polardbx.transaction.tso</td>
<td>TSO 服务相关代码</td>
</tr>
</table>

# 如何入手

PolarDB-X 是一个复杂的系统，代码、接口、模块众多，阅读代码需要一些技巧，推荐先整体后局部的方式

## 整体了解

![CN](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/4239/1636436532234-af1727fd-0eb4-42a5-a0df-449642e25a0b.png)

和所有 SQL 数据库一样，CN 可以划分为协议层、优化器和执行器三层，阅读代码可以首先从每层的输入输出入手，了解从用户发起读写请求到收到结果的整体流程。下面简单介绍各层的一些关键接口

### 协议层

协议层实现了 [MySQL 协议](https://dev.mysql.com/doc/internals/en/client-server-protocol.html)，负责建立连接，接收用户发送的数据包，组装成 SQL 和参数传递给优化器。根据功能，协议层代码可以分为连接管理，数据包解析和协议解析三部分。连接管理和数据包解析代码在 [polardbx-net](https://github.com/ApsaraDB/galaxysql/tree/main/polardbx-net) 模块中，协议解析代码在 [polardbx-server](https://github.com/ApsaraDB/galaxysql/tree/main/polardbx-server/src/main/java/com/alibaba/polardbx/server) 模块中。

-   连接管理的代码可以从建立连接入手了解，入口在 [NIOAcceptor#accept](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/NIOAcceptor.java#L111)。
-   数据包解析是将数据转换为协议数据对象的过程，推荐从文本协议入手了解，入口在 [AbstractConnection#read](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/AbstractConnection.java#L186)
-   协议解析是将协议数据对象分发到具体执行逻辑的过程，入口在 [FrontendCommandHandler#handle](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/handler/FrontendCommandHandler.java#L45)

### 优化器

对 SQL 的处理包括语法解析，校验(Validate)，生成逻辑计划，逻辑计划优化，物理计划优化五个步骤，优化产出物理执行计划，传入执行器。优化器使用了 [Apache Calcite](https://calcite.apache.org/) 的 RBO/CBO 框架，因此优化器框架代码在 [polardbx-calcite](https://github.com/ApsaraDB/galaxysql/tree/main/polardbx-calcite) 模块中，具体实现在 [polardbx-optimizer](https://github.com/ApsaraDB/galaxysql/tree/main/polardbx-optimizer) 模块中。优化器入口在 [Planner#plan](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/Planner.java#L256)，以下列出各个步骤的关键接口

<table>
<tr class="header">
<th>步骤</th>
<th>接口</th>
</tr>
<tr class="odd">
<td>语法解析</td>
<td><a href="https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/parse/FastsqlParser.java#L79">FastsqlParser#parse</a></td>
</tr>
<tr class="even">
<td>校验</td>
<td><a href="https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/SqlConverter.java#L162">SqlConverter#validate</a></td>
</tr>
<tr class="odd">
<td>逻辑计划生成</td>
<td><a href="https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/SqlConverter.java#L188">SqlConverter#toRel</a></td>
</tr>
<tr class="even">
<td>逻辑计划优化</td>
<td><a href="https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/Planner.java#L942">Planner#optimizeBySqlWriter</a></td>
</tr>
<tr class="odd">
<td>物理计划优化</td>
<td><a href="https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/Planner.java#L971">Planner#optimizeByPlanEnumerator</a></td>
</tr>
</table>

### 执行器

执行器接收到物理执行计划后，首先根据计划类型确定执行模式，包括 cursor / local / mpp 三种执行模式。不同行模式下每个算子都对应的执行代码可能有差异，因此还需要将算子绑定到执行代码。执行过程会通过 RPC 接口与 DN 通信，下发读写请求并汇总结果。以下列出关键接口

<table>
<tr class="header">
<th>步骤</th>
<th>接口</th>
</tr>
<tr class="odd">
<td>执行器入口</td>
<td><a href="https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/PlanExecutor.java#L49">PlanExecutor#execute</a></td>
</tr>
<tr class="even">
<td>执行模式选择</td>
<td><a href="https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/ExecutorHelper.java#L78">ExecutorHelper#execute</a></td>
</tr>
<tr class="odd">
<td>cursor 模式算子绑定到执行代码</td>
<td><a href="https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/AbstractGroupExecutor.java#L70">AbstractGroupExecutor#executeInner</a></td>
</tr>
<tr class="even">
<td>local 模式算子绑定到执行代码</td>
<td><a href="https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/mpp/operator/LocalExecutionPlanner.java#L312">LocalExecutionPlanner#plan</a></td>
</tr>
<tr class="odd">
<td>mpp 模式切分执行计划</td>
<td><a href="https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/mpp/planner/PlanFragmenter.java#L109">PlanFragmenter.Fragmenter#buildRootFragment</a></td>
</tr>
<tr class="even">
<td>cursor 模式与 DN 通信</td>
<td><a href="https://github.com/ApsaraDB/galaxysql/blob/main/polardbx-executor/src/main/java/com/alibaba/polardbx/repo/mysql/spi/MyJdbcHandler.java">MyJdbcHandler</a></td>
</tr>
<tr class="odd">
<td>local/mpp 模式与 DN 通信</td>
<td><a href="https://github.com/ApsaraDB/galaxysql/blob/main/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/operator/TableScanClient.java">TableScanClient</a></td>
</tr>
</table>

## 深入了解

要深入了解模块代码，比较好的方式应该是带着问题去看。比如想要了解协议层代码，首先思考最简单的查询 `select 1` 的处理流程是怎样的，结合前面的模块和接口介绍，跟踪调试后不难得到答案，然后继续思考其他语句/协议类型（比如 SET 和 Prepared Statement）处理起来有何不同？收到/返回的包过大如何处理？ssl 如何处理？等等。另外要注意，虽然是深入了解，依然推荐在阅读过程中分清主次，“目录与模块” 小节中列出的包建议重点关注，其它包中的内容则可以先快速带过。

# 小结

本文主要介绍 CN (Compute Node) 代码，涉及 GalaxySQL 和 GalaxyRpc 两个仓库，目的是帮助读者快速掌握 CN 代码的整体结构。从功能上看，CN 完成三个任务：协议处理、查询优化、与 DN 交互，代码也因此可以分为协议层、优化器、执行器三部分。文章首先介绍了代码工程的组织形式，列出编译调试相关文档。然后展开各个目录模块对应的功能，便于需要深入了解代码的读者快速定位。最后给出代码阅读建议，并列举各个模块中的关键接口，供读者调试使用。



Reference:

