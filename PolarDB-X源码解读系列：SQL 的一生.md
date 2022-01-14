[前文](https://zhuanlan.zhihu.com/p/440865170)中，我们给出了组成 GalaxySQL 的三个部分，并从目录入手介绍了重要模块/目录，最后不加解释的列出了一些关键接口作为调试代码的切入点。本文将从 SQL 执行角度出发，介绍 GalaxySQL(CN) 代码中与 SQL 解析执行相关的关键代码。

# 概述

“SQL 的一生” 特指从客户端创建连接发送 SQL 开始，到客户端收到返回结果结束，期间 CN 代码中发生的故事。与人的一生类似，从不同角度观测 “SQL 的一生”，有不同的结论，比如：

1.  如果把 CN 看成是一个支持 MySQL 协议的网络程序，“SQL 的一生”代表：接收用户连接请求，建立上下文，接收包含 SQL 的数据包，内部处理后按照 MySQL 协议组装数据包返回给客户端
1.  如果把 CN 看成是一个 SQL 解释器，“SQL 的一生”代表：接收 SQL 文本，经过词法解析、语法解析、语义解析得到逻辑计划，优化逻辑计划得到物理计划，执行物理计划得到最终结果
1.  如果把 CN 看成是一个任务执行引擎，“SQL 的一生”代表：接收用户请求，转换为执行计划，确定调度策略，执行任务，返回结果

![architecture](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/4239/1639738156246-16a1152b-c547-4103-87b8-d5a05b7f0ba1.png)

如上图所示，从整体上看，CN 可以分为协议层、优化器、执行器三部分。“SQL 的一生”从 协议层开始，协议层负责接受用户连接请求，建立连接上下文，将用户发来的数据包转换为 SQL 语句，交给优化器生成物理执行计划。物理执行计划中包含本地执行的算子和下发给 DN 的物理 SQL，执行器首先下发物理 SQL 到 DN，然后汇总 DN 返回的结果交给本地执行的算子处理，最后将处理结果返回给协议层，按照 MySQL 协议封装成数据包后发送给用户。

> 由于篇幅关系，本文仅包含 GalaxySQL(CN) 相关内容，GalaxyEngine(DN) 相关内容留在后续文章中介绍


下文中，会展开介绍协议层、优化器、执行器三部分在 SQL 解析执行过程的实现细节。

# 协议层

协议层首先要完成的工作是监听网络端口，等待用户建立连接，[CN 启动流程](https://zhuanlan.zhihu.com/p/443608658) 中介绍过，这部分逻辑在 [NIOAcceptor](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/NIOAcceptor.java#L53) 的构造函数中，每个 CN 进程只启动一个 NIOAcceptor 用于监听网络端口。

![NIOAcceptor#accept](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/4239/1640030418757-5175922f-b40c-498d-8071-464e027f3020.png)

收到客户端发起的 TCP 连接请求后，协议层在 [NIOAcceptor#accept](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/NIOAcceptor.java#L111) 中将 TCP 连接绑定到一个 NIOProcessor 上，每个 NIOProcessor 会启动两个线程，分别用于读/写 TCP 数据包，读/写线程的实现封装在 [NIOReactor](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/NIOReactor.java#L74) 中。

同时，[NIOAcceptor#accept](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/NIOAcceptor.java#L111) 中还将 NIOProcessor 和一个 FrontendConnection 对象绑定在一起。FrontendConnection 中封装了 MySQL 协议处理逻辑和 Session Context 相关信息，有 [ServerConnection](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-server/src/main/java/com/alibaba/polardbx/server/ServerConnection.java#L273) 和 [ManagerConnection](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-server/src/main/java/com/alibaba/polardbx/manager/ManagerConnection.java#L38) 两个具体实现，ServerConnection 用于处理客户端发送的 SQL，ManagerConnection 用于处理一些内部管理命令。

ServerConnection 中解析 MySQL 协议数据包的逻辑封装在 NIOHandler 中，有 [FrontendAuthorityAuthenticator](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/handler/FrontendAuthorityAuthenticator.java#L54) 和 [FrontendCommandHandler](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/handler/FrontendCommandHandler.java#L45) 两个实现，分别用于鉴权和命令执行。具体过程是，[NIORreactor](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/NIOReactor.java#L132) 调用 [AbstractConnection#read](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/AbstractConnection.java#L186) 完成 ssl 解析、大包重组等操作，得到具体要处理的数据包，然后调用 [FrontendConnection#handleData](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/FrontendConnection.java#L666) 从线程池中获取一个新的线程，最后调用 [FrontendConnection#handler#handle](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/FrontendConnection.java#L746) 接口完成鉴权和数据包解析。

![FrontendCommandHandler#handle](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/4239/1640035805762-d60eb79e-5b0a-41ab-85ec-26abf977ff90.png)

如上图所示，[FrontendCommandHandler#handle](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/handler/FrontendCommandHandler.java#L45) 中根据 Command 类型，调用不同的处理函数，最常用的 Command 是 [COM_QUERY](https://dev.mysql.com/doc/internals/en/com-query.html#packet-COM_QUERY)，所有 DML / DDL 语句，只要没有通过 PreparedStatement 执行，Command 都是 COM_QUERY，对应的处理函数是 [FrontendConnection#query](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/FrontendConnection.java#L449)。

[FrontendConnection#query](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-net/src/main/java/com/alibaba/polardbx/net/FrontendConnection.java#L449) 方法继续解析数据包，得到 SQL 文本，传入 [ServerQueryHandler#queryRaw](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-server/src/main/java/com/alibaba/polardbx/server/ServerQueryHandler.java#L71) 根据 SQL 种类分类执行，如果客户端发送的是 [MySQL Multi-Statement](https://dev.mysql.com/doc/internals/en/multi-statement.html) 那么这里会将按照语法将多语句切开，分别下发执行。另外这里有个细节是 [ServerQueryHandler#executeStatement](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-server/src/main/java/com/alibaba/polardbx/server/ServerQueryHandler.java#L116) 方法会为每个 SQL 语句生成一个唯一的 `traceId`，`traceId` 最终会出现在各种日志文件和实际下发的物理 SQL 的 HINT 中（从 DN 的 binlog 中可以获取到），用来排查问题十分方便。

常见的 DML / DDL 语句的处理逻辑封装在 [ServerConnection#innerExecute](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-server/src/main/java/com/alibaba/polardbx/server/ServerConnection.java#L850) 方法中，处理过程可以分为优化执行和返回数据包两部分。[TConnection#executeQuery](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-server/src/main/java/com/alibaba/polardbx/matrix/jdbc/TConnection.java#L521) 中封装了优化执行部分的逻辑，其中 [Planner#plan](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-server/src/main/java/com/alibaba/polardbx/matrix/jdbc/TConnection.java#L531) 为优化器入口，[PlanExecutor#execute](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-server/src/main/java/com/alibaba/polardbx/matrix/jdbc/TConnection.java#L632) 为执行器入口。

执行结果包装成 ResultCursor 对象，在 [ServerConnection#sendSelectResult](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-server/src/main/java/com/alibaba/polardbx/server/ServerConnection.java#L2505) 中读取结果，组装成 [COM_QUERY Response](https://dev.mysql.com/doc/internals/en/com-query-response.html) 数据包返回给客户端。

执行过程中，[ExecutionContext](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/context/ExecutionContext.java) 作为 Session Context 记录了执行计划属性 / SQL 参数 / HINT 等上下文信息，在三个模块之间传递。

# 优化器

和所有数据库一样，优化器模块的主干流程中包含 Parser / Validator / SQL Rewriter(RBO) / Plan Enumerator(CBO) 四个基础步骤，在此之上 GalaxySQL 结合自身特点增加了三个步骤:

-   Plan Management：包含 Plan Cache 和执行计划演进，用于消除执行计划生成带来的额外开销，和通过灰度演进避免 CBO 版本升级导致查询性能回退
-   Mpp Planner：在 Plan Enumerator 之后增加一个阶段，针对 MPP 模式的执行计划，基于数据分区情况，减少 shuffle
-   Post Planner：结合参数进行分区裁剪，根据计划中每张表的裁剪结果继续执行下推，用于处理因为参数化无法进一步优化下推的场景

![optimizer](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/4239/1640058237840-d30b74d5-6eab-432a-b8cf-47b4fd6d1e5b.png)

SQL 在优化器中需要经过 `Parser` --> `Plan Management` --> `Validator` --> `SQL Rewriter(RBO)` --> `Plan Enumerator(CBO)` --> `Mpp Planner` --> `Post Planner` 七步处理，入口为 [Planner#plan](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/Planner.java#L256) ，以下将展开介绍各个步骤中的关键接口。

## Parser

Parser 实现基于阿里巴巴开源的连接池管理软件 [Druid](https://github.com/alibaba/druid) 中的 Parser 组件，是一个手工编写的解析器，用于将 SQL 文本转换为抽象语法树(AST)。Parser 本身包含用于词法解析的 [MySqlLexer](https://github.com/ApsaraDB/galaxysql/blob/f5bc69c7916c4c2b907e0acaf8b8d81ca55eac44/polardbx-parser/src/main/java/com/alibaba/polardbx/druid/sql/dialect/mysql/parser/MySqlLexer.java) 和语法解析的 [MySqlStatementParser](https://github.com/ApsaraDB/galaxysql/blob/f5bc69c7916c4c2b907e0acaf8b8d81ca55eac44/polardbx-parser/src/main/java/com/alibaba/polardbx/druid/sql/dialect/mysql/parser/MySqlStatementParser.java)，为何这样划分超出了本文的范围，通过一个例子简单说明下。与我们阅读一句话类似，解析 SQL 时首先需要将 SQL 文本切分为多个“单词”(Token)，比如一条简单的查询语句

```sql
SELECT * FROM t1 WHERE id > 1;
```

会被切分为如下 Token

```sql
SELECT (keyword), * (identifier), FROM (keyword), t1 (identifier), 
WHERE (keword), id (identifier), > (gt), 1 (literal_int), ; (semicolon)
```

这个切分“单词”的过程就是词法解析。接下来，语法解析会按顺序读取所有 Token，判断是否满足某个 SQL 子句语法的同时，生成 AST，SELECT 语句结果解析生成的对象是 [SqlSelect](https://github.com/ApsaraDB/galaxysql/blob/f5bc69c7916c4c2b907e0acaf8b8d81ca55eac44/polardbx-calcite/src/main/java/org/apache/calcite/sql/SqlSelect.java)，如下图所示

![select](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/4239/1640152762345-a03e55bd-1ee4-400e-b04e-3229e76093e0.png)

细心的同学可能注意到了，SqlSelect 本身和其中成员变量的类型 SqlNode 都在 polardbx-calcite 包而不是 polardbx-parser 包中，原因是 GalaxySQL 使用了 [Apache Calcite](https://calcite.apache.org/) 作为优化器框架，整个框架与 AST 数据结构强绑定，因此 SQL 解析过程首先得到 Druid 的 AST 对象，然后通过一个 visitor 转换为 SqlNode。代码在 [FastSqlToCalciteNodeVisitor](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/parse/visitor/FastSqlToCalciteNodeVisitor.java) ，其中也包含了一些语法改写和权限校验

另外，为了更好的支持 Plan Cache，所有 SQL 需要首先进行参数化，也就是使用占位符替换 SQL 文本中的常量参数，将 SQL 文本转换为参数化 SQL + 参数列表，比如，`SELECT * FROM t1 WHERE id > 1` 会被转换为 `SELECT * FROM t1 WHERE id > ?` 和一个参数 1，这样做的好处是相同模版不同参数的 SQL 可以命中同一条 Plan Cache，效率更高。当然也有缺点，当参数不同时整个查询的代价也不同，可能需要使用不同的计划，这个问题在 Plan Management 中得到了解决。参数化相关代码位于 [DrdsParameterizeSqlVisitor](https://github.com/ApsaraDB/galaxysql/blob/01216d9f3c919a76f7266f2bf49b807486f3ea27/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/parse/visitor/DrdsParameterizeSqlVisitor.java) ，实现上依然是一个 visitor，返回结果封装在 SqlParameterized 对象中。

## Plan Management & Plan Cache

从数据结构上讲，Plan Cache 可以认为是一个 以 SQL 模版、参数信息、元数据版本等信息为 key，执行计划为 value 的一个 map。用途是减少重复优化相同 SQL 模版带来的性能开销，以及结合执行计划演进消除可能由于版本升级带来的性能回退。关于 Plan Management 的详细介绍可以参考 [PolarDB-X 优化器核心技术 ~ 执行计划管理](https://zhuanlan.zhihu.com/p/398558605)。

调用 Plan Cache 和 Plan Management 的逻辑封装在 [Planner#doPlan](https://github.com/ApsaraDB/galaxysql/blob/01216d9f3c919a76f7266f2bf49b807486f3ea27/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/Planner.java#L1648) 中，Plan Cache 的实现代码在 [PlanCache#get](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/PlanCache.java#L123)，Plan Management 的实现代码在 [PlanManager#choosePlan](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/planmanager/PlanManager.java#L249)。

实际上，Plan Cache 的实现类似 Java 中的 Map.computIfAbsent()：如果 map 中存在已经生成好的执行计划，则直接返回执行计划；如果不存在则生成一个执行计划保存在 map 中然后返回这个执行计划。也就是说经过 Plan Management & Plan Cache 之后一定会拿到一个执行计划，Validator / SQL Rewriter / Plan Enumerator / Mpp Planner 四个步骤其实是在 Plan Cache 内部被调用的，Post Planner 由于依赖具体参数，不能 Cache，需要拿到执行计划之后调用。

## Validator

早期的数据库实现经常在 AST 上直接进行查询优化，但由于 AST 缺少关系代数算子的层次结构，难以写出相互正交的优化规则，会导致所有优化逻辑堆叠在一起，维护困难。现代数据库实现通常都会在 validating 或者 binding 过程中将 AST 转换为关系代数算子组成的算子树作为逻辑计划，GalaxySQL 也不例外。关于实现了哪些算子可以参考 [执行计划介绍
](https://doc.polardbx.com/sql-tunning/topics/sql-plan.html)。

AST 到逻辑计划的转换分为两步，入口在 [Planner#getPlan](https://github.com/ApsaraDB/galaxysql/blob/01216d9f3c919a76f7266f2bf49b807486f3ea27/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/Planner.java#L785)，首先在 [SqlConverter#validate](https://github.com/ApsaraDB/galaxysql/blob/01216d9f3c919a76f7266f2bf49b807486f3ea27/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/Planner.java#L839) 中进行语义检查，包括名字空间校验，类型校验等步骤，比较特别的一个点是 [SqlValidatorImpl#performUnconditionalRewrites](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-calcite/src/main/java/org/apache/calcite/sql/validate/SqlValidatorImpl.java#L1637) 中包含了一些对 AST 改写的内容，主要用于屏蔽相同语义的不同语法结构。然后  [SqlConverter#toRel](https://github.com/ApsaraDB/galaxysql/blob/01216d9f3c919a76f7266f2bf49b807486f3ea27/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/Planner.java#L841) 中包含了将用 SqlNode 对象保存的 AST 转换为使用 RelNode 对象保存的逻辑计划，转换过程比较复杂这里不做展开。

## SQL Rewriter

SQL Rewriter 是 GalaxySQL 的 RBO 组件，工作内容是使用固定规则组对逻辑计划进行优化。GalaxySQL 中 RBO 优化可以分为两个部分，一个是执行传统的关系代数优化（比如 谓词推导、LEFT JOIN 转 INNER JOIN 等），另一个是完成部计算下推工作（更多内容参考 [PolarDB-X 优化器核心技术 ~ 计算下推](https://zhuanlan.zhihu.com/p/366312701)）。

调用 RBO 的代码封装在 [Planner#optimizeBySqlWriter](https://github.com/ApsaraDB/galaxysql/blob/01216d9f3c919a76f7266f2bf49b807486f3ea27/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/Planner.java#L941)，RBO 框架基于 [HepPlanner](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-calcite/src/main/java/org/apache/calcite/plan/hep/HepPlanner.java#L187)，实现了大量优化规则，规则分组信息保存在 [RuleToUse](https://github.com/ApsaraDB/galaxysql/blob/01216d9f3c919a76f7266f2bf49b807486f3ea27/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/rule/RuleToUse.java#L87) 中，规则组执行的先后顺序记录在 [SQL_REWRITE_RULE_PHASE](https://github.com/ApsaraDB/galaxysql/blob/01216d9f3c919a76f7266f2bf49b807486f3ea27/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/rule/SQL_REWRITE_RULE_PHASE.java) 中。

![PushJoinRule](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/4239/1640329321531-4c24c336-a379-4f64-b612-580af9f8d217.png)

RBO 规则要实现三个固定内容，匹配的子树结构、优化逻辑、返回变换后的计划。上图展示的是用于下推 JOIN 的规则，70-72 行在构造函数中指明这个规则匹配的是 LogicalJoin 下挂两个 LogicalView 的算子树，当 RBO 框架匹配到这种子树时会调用规则的 onMatch 接口，规则完成优化后，调用 [RelOptRuleCall.transformTo](https://github.com/ApsaraDB/galaxysql/blob/a66b025c7ef8d9b20c3382d8b05bccca2abee9d4/polardbx-calcite/src/main/java/org/apache/calcite/plan/hep/HepRuleCall.java#L55) 返回变换后的计划。

RBO 框架按照组内乱序执行，组间串行执行的方式执行 [SQL_REWRITE_RULE_PHASE](https://github.com/ApsaraDB/galaxysql/blob/01216d9f3c919a76f7266f2bf49b807486f3ea27/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/rule/SQL_REWRITE_RULE_PHASE.java) 列出的所有规则，每条规则都会反复匹配，直到某一轮匹配后，一组内的规则全都没有命中，则认为该组执行结束。

## Plan Enumerator

Plan Enumerator 是 GalaxySQL 的 CBO 组件，工作是生成/枚举物理执行计划，并根据代价选出最合适的物理执行计划，具体包括 Join Reorder、索引选择、物理执行计划选择等，详细介绍参考 [PolarDB-X CBO 优化器技术内幕](https://zhuanlan.zhihu.com/p/370372242)。

调用 CBO 的代码封装在 [Planner#optimizeByPlanEnumerator](https://github.com/ApsaraDB/galaxysql/blob/2599de6d01f5a9953431f8b1f2132fec745e2be4/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/Planner.java#L971) 中，CBO 框架基于 [VolcanoPlanner](https://github.com/ApsaraDB/galaxysql/blob/main/polardbx-calcite/src/main/java/org/apache/calcite/plan/volcano/VolcanoPlanner.java) 框架，与 RBO 不同，CBO 规则无需提前分组，所有用到的规则都保存在 [RuleToUse#CBO_BASE_RULE](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/rule/RuleToUse.java#L408) 中。与 RBO 规则相同，CBO 规则的执行流程也是匹配一颗子树然后返回变换后的计划，区别在于 CBO 不会直接用新生成的计划替换原来的子树，而是将生成的新的执行计划保存在 RelSubset 中，后续从 RelSubset 中选出代价最低的一个计划，代码入口在[VolcanoPlanner#findBestExp](https://github.com/ApsaraDB/galaxysql/blob/2599de6d01f5a9953431f8b1f2132fec745e2be4/polardbx-calcite/src/main/java/org/apache/calcite/plan/volcano/VolcanoPlanner.java#L511)。

## Mpp Planner

Mpp Planner 是 GalaxySQL 针对 MPP 执行计划增加的优化阶段，负责根据数据分布生成 exchange 算子，减少冗余的数据 shuffle。代码入口在[Planner#optimizeByMppPlan](https://github.com/ApsaraDB/galaxysql/blob/2599de6d01f5a9953431f8b1f2132fec745e2be4/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/Planner.java#L1196)，生成 MppExchange 算子的代码在 [MppExpandConversionRule#enforce](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/rule/mpp/MppExpandConversionRule.java#L67)。

## Post Planner

Post Planner 主要用于带入实际参数进行分区裁剪后，根据下发的分片情况二次下推执行计划。假设表 r 和表 t 在一个 table group 中，拆分字段都是 id，考虑下面的 SQL

```sql
SELECT * FROM r JOIN t ON r.name = t.name WHERE r.id = 0 AND t.id = 1;
```

RBO / CBO 中看到的是参数化之后的计划，类似 `SELECT * FROM r JOIN t ON r.name = t.name WHERE r.id = ? AND t.id = ?;` 由于没有分区键上的相等条件，无法确定所需的数据是否落在相同的 partition group 上，但是结合参数进行分区裁剪之后，就会发现其实数据都落在 0 号分片上，是一条单分片 Join 语句，可以直接下发。

PostPlanner 在 [Planner](https://github.com/ApsaraDB/galaxysql/blob/01216d9f3c919a76f7266f2bf49b807486f3ea27/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/Planner.java#L1666) 中被调用，实现代码在 [PostPlanner#optimize](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/PostPlanner.java#L144) 中。

# 执行器

![executor](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/4239/1641529443931-63c9c86b-2c6c-41fe-9c62-c1ff5e7348ea.png)

PolarDB-X 执行器支持两种执行模型，传统的 Volcano 迭代模型和支持 Pipeline 向量化的 push 模型，两种模型各有优劣详细介绍参考[PolarDB-X 面向 HTAP 的混合执行器](https://zhuanlan.zhihu.com/p/337062566)。执行计划进入执行器后存在三条可选的执行链路： Cursor、 Local、Mpp。Cursor 链路用于执行 DML/DDL/DAL 语句，Local、Mpp 链路用于执行 DQL 语句。Cursor 链路仅支持 Volcano 迭代模型，Local 链路同时支持两种模型，Mpp 链路在 Local 链路的基础上增加了多机任务调度。

执行器入口代码在 [PlanExecutor#execByExecPlanNodeByOne](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/PlanExecutor.java#L90) ，之后在 [ExecutorHelper#execute](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/ExecutorHelper.java#L78) 中根据优化器确定的执行模式选择执行链路。以下以 Cursor 链路为例介绍计划执行的整体流程

![cursor](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/4239/1641543529462-8e39ad20-f330-4bbc-bca5-b2de982746f3.png)

Cursor 链路中，首先根据执行计中的算子找到对应的 handler，代码位置在[CommandHandlerFactoryMyImp#getCommandHandler](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/repo/mysql/handler/CommandHandlerFactoryMyImp.java#L442)，handler 负责将当前算子转换为一个 [Cursor](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/cursor/Cursor.java) 接口的具体实现，并且嵌套的调用 [CommandHandlerFactoryMyImp#getCommandHandler](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/repo/mysql/handler/CommandHandlerFactoryMyImp.java#L442) 接口将整个执行计划转换为一颗Cursor 树。[ResultSetUtil#resultSetToPacket](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-server/src/main/java/com/alibaba/polardbx/server/executor/utils/ResultSetUtil.java#L142) 中调用 Cursor.next 接口获取全部执行结果返回给用户。

![Executor](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/4239/1641545782624-8cb1ebf4-ddad-4672-a0b7-a47dd364d107.png) ![ConsumerExecutor](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/4239/1641545743827-66357841-887d-4652-8a16-cf66fb3b9dd3.png)

Local 链路的执行入口在 [SqlQueryLocalExecution#start](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/mpp/execution/SqlQueryLocalExecution.java#L135) ，首先在 [LocalExecutionPlanner#plan](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/mpp/operator/LocalExecutionPlanner.java#L312) 中将执行计划切分为多个 Pipeline，切分的具体逻辑在 [LocalExecutionPlanner#visit](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/mpp/operator/LocalExecutionPlanner.java#L371) ，然后为每个 Pipeline 生成一个包含具体执行逻辑的 Driver 放入调度队列，执行逻辑封装在 [Executor](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/operator/Executor.java) 或者 [ConsumerExecutor](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/operator/ConsumerExecutor.java) 中，分别代表 迭代模型和 push 模型。执行结果封装在 [SmpResultCursor](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/mpp/client/SmpResultCursor.java) 中，与 Cursor 相同，[ResultSetUtil#resultSetToPacket](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-server/src/main/java/com/alibaba/polardbx/server/executor/utils/ResultSetUtil.java#L142) 中调用 Cursor.next 接口获取全部执行结果返回给用户。

Mpp 链路的执行入口在 [SqlQueryExecution#start](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/mpp/execution/SqlQueryExecution.java#L123) ，首先在 [PlanFragmenter#buildRootFragment](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/mpp/planner/PlanFragmenter.java#L109) 中根据 MppPlanner 插入的 Exchange 算子将执行计划切分成多个 Fragment ，之后在 [SqlQueryScheduler#schedule](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/mpp/execution/scheduler/SqlQueryScheduler.java#L409) 中调用 [StageScheduler](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/mpp/execution/scheduler/StageScheduler.java) 生成和下发并行计算任务，任务信息封装在 [HttpRemoteTask](https://github.com/ApsaraDB/galaxysql/blob/4ba7ff422dc63d8e95f82c4e319c4ece692622ea/polardbx-executor/src/main/java/com/alibaba/polardbx/executor/mpp/server/remotetask/HttpRemoteTask.java) 对象中，并行计算任务下发到执行节点后的执行链路与 Local 链路相同。关于 Mpp 链路的更多原理介绍请参考 [PolarDB-X 并行计算框架](https://zhuanlan.zhihu.com/p/346320114)

# 小结

本文从 SQL 执行的角度出发，介绍了协议层、优化器、执行器的关键代码。协议层负责网络连接管理、数据包到 SQL 和执行结果到数据包的转换，第一小节介绍协议层的执行流程，并给出了基于 NIO 的网络连接管理、SSL 协议解析、鉴权、协议解析、大包重组等内容的代码入口。优化器负责 SQL 到执行计划的转换，转换过程包含七个阶段 `Parser` --> `Plan Management` --> `Validator` --> `SQL Rewriter(RBO)` --> `Plan Enumerator(CBO)` --> `Mpp Planner` --> `Post Planner`，第二小节简单介绍了每个阶段的工作内容并给出了代码入口。执行器负责根据执行计划获得最终执行结果，支持迭代模型和 Pipeline 向量化模型，分为 Cursor、Local 和 MPP 三条执行链路，第三小节介绍了各个链路的基本流程和关键入口。



Reference:

