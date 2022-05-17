-   TODO
    -   补全整体流程图的箭头
    -   元信息同步过程须补充元信息版本检查及并发DDL可能引入的问题
    -   DML的重写过程须补充分布式事务

## 概述

一条SQL语句进入PolarDB-X的CN后，将经历协议层、优化器、执行器的完整处理流程。首先经过解析、鉴权、校验，被解析为关系代数树后，在优化器中经历RBO和CBO生成执行计划，最终在DN上执行完成。与DML不同的是，逻辑DDL语句还涉及对元数据的读写和物理DDL，直接影响系统状态一致性。

PolarDB-X的DDL实现的关键目标是DDL的“**online**”和“**crash safe**”，即DDL与DML的并发和DDL本身的原子性和持久性。对于复杂逻辑DDL（如加减全局二级索引、迁移分区数据），PolarDB-X沿袭online schema change的思路引入了CN中的双版本元数据，DDL仅在排空低版本事务时占用MDL锁，大大降低了阻塞业务SQL的频率；在执行器模块中，DDL引擎统一了逻辑DDL的定义方法和调度执行过程，开发者只须将物理操作封装为具有时序依赖关系和crash recover方法的Task传递给DDL引擎，后者保证相应的执行和回滚逻辑。

本文主要解读 PolarDB-X  DDL在计算节点（CN）中的实现，为了简便起见，本文中我们将略过DDL在协议层、优化器的预处理以及执行器中对Handler的分派过程，此部分内容可参考下列源码解读文章：

-   [PolarDB-X SQL 的一生](https://zhuanlan.zhihu.com/p/457450880)
-   [DML 之 Insert 流程](https://zhuanlan.zhihu.com/p/466296747)

我们将重点关注DDL在执行器中的执行流程，在阅读本文前，可预先阅读与PolarDB-X online schema change、元数据锁及DDL引擎原理相关的文章：

-   [PolarDB-X Online Schema Change](https://zhuanlan.zhihu.com/p/341685541)
-   [PolarDB-X：让“Online DDL”更Online](https://zhuanlan.zhihu.com/p/347885003)
-   [PolarDB-X DDL也要追求ACID](https://zhuanlan.zhihu.com/p/435352604)

## 整体流程

![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/308295/1645778806739-27478c2f-0f0e-4b99-9b95-5096ad955abb.png#clientId=u927355fc-f071-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=1664&id=u14d53bf4&margin=%5Bobject%20Object%5D&name=image.png&originHeight=3328&originWidth=4556&originalType=binary&ratio=1&rotation=0&showTitle=false&size=883829&status=done&style=none&taskId=u6c84ab60-331d-4c5d-8d1b-e1d44547ad6&title=&width=2278)

1.  一条逻辑DDL语句在解析后进入优化器，仅做简单的类型转化后生成logicalPlan及executionContext，由执行器根据logicalPlan的具体类型分派给对应的DDLHandler。DDLHandler的公共基类是`LogicalCommonDdlHandler`，其公共执行入口见`com.alibaba.polardbx.executor.handler.ddl.LogicalCommonDdlHandler#handle`。

```java
    public Cursor handle(RelNode logicalPlan, ExecutionContext executionContext) {
        BaseDdlOperation logicalDdlPlan = (BaseDdlOperation) logicalPlan;

        initDdlContext(logicalDdlPlan, executionContext);

        // Validate the plan first and then return immediately if needed.
        boolean returnImmediately = validatePlan(logicalDdlPlan, executionContext);

        .....
        setPartitionDbIndexAndPhyTable(logicalDdlPlan);
        // Build a specific DDL job by subclass that override buildDdlJob
        DdlJob ddlJob = returnImmediately?
            new TransientDdlJob():
            // @override
            buildDdlJob(logicalDdlPlan, executionContext);

        // Validate the DDL job before request.
        // @override
        validateJob(logicalDdlPlan, ddlJob, executionContext);

        // Handle the client DDL request on the worker side.
        handleDdlRequest(ddlJob, executionContext);
        .....
        return buildResultCursor(logicalDdlPlan, executionContext);
    }
```

1.  在`handle`方法中，首先从executionContext中剥离出traceId、事务ID、ddl类型、全局配置等通用上下文信息，然后通过`validateJob`方法校验DDL正确性，`buildDdlJob`方法构造DDL任务。具体的DDLHandler例如`AlterTableHandler`将会重载这两个方法，由开发者根据DDL的实际语义定义和校验DDL job。
1.  至此，DDL在handler中完成从logicalPlan到DDL job的转化。`com.alibaba.polardbx.executor.handler.ddl.LogicalCommonDdlHandler#handleDdlRequest`方法将DDL job对应的DDL Request转发给Leader CN节点上的DDL引擎调度执行。
1.  DDL引擎作为守护进程仅在Leader CN节点运行，轮询DDLJobQueue，队列非空即触发DDL Job的调度执行过程。DDL引擎中维持着两级队列，即实例级和库级的DDLJob队列，从而保证逻辑库间DDL并发执行，互不干扰。

值得注意的是，在DDL Requester将DDL Request推送至DDL引擎的过程中，实际上发生了一次Worker Node到Leader Node的**节点间通信**，该过程依赖于`com.alibaba.polardbx.gms.sync.GmsSyncManagerHelper#sync`方法。发起通信的节点在sync中封装syncAction作为信息，接收方监听到sync发生后根据syncAction执行相应的动作。这种节点间通信机制不仅存在于DDL引擎加载DDL job过程中，也存在于DDL job执行时节点间的元数据同步过程中（见后文），这两种情境下封装的syncAction均会触发接收方从MetaDB中拉取实际的通信内容（DDL job或者元信息），这种通信机制以数据库服务作为通信中介保证了通信内容传递的高可用和一致性。

DDL job是DDL执行引擎的核心概念，它实际上是DDL引擎层面模拟的“**DDL事务**”，尝试对元信息修改、节点间通信、物理DDL、DML等一系列异质操作组合而成的逻辑DDL实现原子性和持久性，从而保证系统状态的稳定性。借鉴于通用的事务实现，**undolog、锁、WAL、版本快照**等事务要素在DDL job和DDL引擎中也有着平行的实现。

以DDL引擎入口（即前述的`handleDdlRequest`方法）作为DDL数据流的分界点，可将DDL的生命周期划分为定义和执行两个部分。接下来，我们将以添加全局二级索引为例，说明在定义DDL Job时构建DDL原子性以及DDL与DML协同流程的关键逻辑。DDL在DDL引擎中的调度执行及错误处理机制，将在下篇中讲解。

```sql
## 本文涉及的SQL语句
####

create database db1 mode = "auto";

use db1;

create table t1(x int, y int);

## 本文详解的DDL语句

alter table t1 add global index `g_i_y` (`y`) COVERING (`x`) partition by hash(`y`);
```

## 元数据管理

如前所述，除去在DN上执行的物理DDL和DML外，DDL Job中包含的关键操作均是对元数据的修改和同步。

PolarDB-X中的元数据由GMS统一管理，按物理位置可以分为两个部分：

1.  MetaDB中的持久化元信息。例如描述分区表t1的元数据表包括`table_partitions, tables, columns, indexes`等，MetaDB中元信息的读写接口分布在`polardbx-gms/src/main/java/com/alibaba/polardbx/gms`路径下的辅助类中。
1.  CN缓存的内存元信息。出于性能考虑，CN节点同时在内存中缓存有分区表t1的元信息，该信息是直接被CN上的DML事务使用的元信息。`com.alibaba.polardbx.executor.gms`中实现了不同种类元信息的缓存管理，其接口定义为`com.alibaba.polardbx.optimizer.config.table.SchemaManager`, 包含`init, getTable, reload`等基本方法。

```java
    private ResultCursor executeQuery(ByteString sql, ExecutionContext executionContext,
                                      AtomicBoolean trxPolicyModified) {

        // Get all meta version before optimization
        final long[] metaVersions = MdlContext.snapshotMetaVersions();

        // Planner
        ExecutionPlan plan = Planner.getInstance().plan(sql, executionContext);

        ...
        //for requireMDL transaction
        if (requireMdl && enableMdl) {
            if (!isClosed()) {
                // Acquire meta data lock for each statement modifies table data
                acquireTransactionalMdl(sql.toString(), plan, executionContext);
            }
            ...
            //update Plan if metaVersion changed, which indicate meta updated
        }
        ...

        ResultCursor resultCursor = executor.execute(plan, executionContext);
        ...
        return resultCursor;
    }


```

值得注意的是，内存元信息以逻辑表为基本单位构建锁，此锁即为**内存MDL锁**，DML事务开始时即通过`acquireTransactionalMdl`尝试获取当前最新版本的相关逻辑表的MDL锁，执行完成后释放。

DDL Job执行时，将按一定的时序修改上述两部分元信息，此过程中DDL和DML之间的同步是实现DDL的online特性的关键步骤。

## DDL Job定义

添加全局二级索引的DDL Job Handler为`AlterTableHandler`，它重载了`LogicalCommonDdlHandler`的buildDdlJob方法`com.alibaba.polardbx.executor.handler.ddl.LogicalAlterTableHandler#buildDdlJob`，该方法最终分派到`com.alibaba.polardbx.executor.ddl.job.factory.gsi.CreatePartitionGsiJobFactory`构造全局二级索引对应的ddl Job。

```java
public class CreatePartitionGsiJobFactory extends CreateGsiJobFactory{
    @Override
    protected void excludeResources(Set<String> resources) {
        super.excludeResources(resources);
        //meta data lock in MetaDB
        resources.add(concatWithDot(schemaName, primaryTableName)); //db1.t1
        resources.add(concatWithDot(schemaName, indexTableName)); //db1.g_i_y
        ...
    }    

    @Override
    protected ExecutableDdlJob doCreate() {
      ...
        if (needOnlineSchemaChange) {
            bringUpGsi = GsiTaskFactory.addGlobalIndexTasks(
                schemaName,
                primaryTableName,
                indexTableName,
                stayAtDeleteOnly,
                stayAtWriteOnly,
                stayAtBackFill
            );
        }
      ...
        List<DdlTask> taskList = new ArrayList<>();
        //1. validate
        taskList.add(validateTask);
        //2. create gsi table
        //2.1 insert tablePartition meta for gsi table
        taskList.add(createTableAddTablesPartitionInfoMetaTask);
        //2.2 create gsi physical table
        CreateGsiPhyDdlTask createGsiPhyDdlTask =
            new CreateGsiPhyDdlTask(schemaName, primaryTableName, indexTableName, physicalPlanData);
        taskList.add(createGsiPhyDdlTask);
        //2.3 insert tables meta for gsi table
        taskList.add(addTablesMetaTask);
        taskList.add(showTableMetaTask);
        //3. 
        //3.1 insert indexes meta for primary table
        taskList.add(addIndexMetaTask);
        //3.2 gsi status: CREATING -> DELETE_ONLY -> WRITE_ONLY -> WRITE_REORG -> PUBLIC
        taskList.addAll(bringUpGsi);
        //last tableSyncTask
        DdlTask tableSyncTask = new TableSyncTask(schemaName, indexTableName);
        taskList.add(tableSyncTask);

        final ExecutableDdlJob4CreatePartitionGsi result = new ExecutableDdlJob4CreatePartitionGsi();
        result.addSequentialTasks(taskList);
        ....

        return result;
    }
    ...
}
```

`excludeResources`方法声明了ddlJob对相关对象的**持久化元数据锁**的占用，本例涉及主表与GSI两张表的元数据修改，因而加锁对象包括`db1.t1, db1.g_i_y`。注意，该锁与前述的内存MDL锁不同，前者在DDL引擎执行DDL job的初始阶段持久化到MetaDB的read_write_lock表中，用于控制**DDL之间的并发**。
`doCreate`方法声明了按时序执行的一系列Task，其语义依次是：

-   校验
    -   `validateTask`: DDL Job校验，检查GSI表名的合法性等。

-   创建GSI表
    -   `createTableAddTablesPartitionInfoMetaTask` GSI的分区元信息写入
    -   `createGsiPhyDdlTask` 创建GSI对应物理表的物理DDL
    -   `addTablesMetaTask` 主表的元信息修改
    -   `showTableMetaTask` 主表的元信息在节点之间广播

-   GSI表的元信息同步
    -   `addIndexMetaTask` GSI表的tableMeta元信息修改
    -   `bringUpGsi` 添加索引后的online schema change过程
    -   `tableSyncTask`GSI表的元信息在节点之间广播

在上述的Task中，持久化元信息的写入操作均作用于MetaDB中相应的元信息表，元信息的广播操作作用于CN缓存的元信息。

接下来我们简要介绍其中的三类代表性Task，其中`CreateTableAddTablesMetaTask`包含的方法列表如下：

```java
public class CreateTableAddTablesMetaTask extends BaseGmsTask {
    @Override
    public void executeImpl(Connection metaDbConnection, ExecutionContext executionContext) {
        PhyInfoSchemaContext phyInfoSchemaContext = TableMetaChanger.buildPhyInfoSchemaContext(schemaName,
            logicalTableName, dbIndex, phyTableName, sequenceBean, tablesExtRecord, partitioned, ifNotExists, sqlKind,
            executionContext);
        FailPoint.injectRandomExceptionFromHint(executionContext);
        FailPoint.injectRandomSuspendFromHint(executionContext);
        TableMetaChanger.addTableMeta(metaDbConnection, phyInfoSchemaContext);
    }

    @Override
    public void rollbackImpl(Connection metaDbConnection, ExecutionContext executionContext) {
        TableMetaChanger.removeTableMeta(metaDbConnection, schemaName, logicalTableName, false, executionContext);
    }

    @Override
    protected void onRollbackSuccess(ExecutionContext executionContext) {
        TableMetaChanger.afterRemovingTableMeta(schemaName, logicalTableName);
    }
}
```

1.  `addTableMetaTask`在主表的tableMeta中添加GSI表相关的元信息。`executeImpl`调用MetaDB提供的元数据读写接口`com.alibaba.polardbx.executor.ddl.job.meta.TableMetaChanger#addTableMeta`写入GSI表的元信息。在executeImpl对应的主线逻辑之外，`addTableMetaTask`同时提供了`rollbackImpl`方法清空GSI表的元信息，这实际上即是该Task对应的**undolog**，在DDL job发生回滚时调用该方法即可恢复原有的tableMeta信息。
1.  `TableSyncTask`作为前述的**节点间通信机制**的一个特例，实现tableMeta元信息在CN节点间的同步，通知集群内所有CN节点从MetaDB中加载tableMeta元信息。
1.  `bringUpGsi`是一系列Task构成的Task List，它参照online schema change定义了元信息的演化和数据回填流程，其原理见[PolarDB-X Online Schema Change](https://zhuanlan.zhihu.com/p/341685541)，其中也包含TableSyncTask。我们将在下一节着重介绍此Task.

除了上述Task，在逻辑DDL中常用的元信息读写、物理DDL和DML Task均已预先定义，读者可在`polardbx-executor/src/main/java/com/alibaba/polardbx/executor/ddl/job/task/basic`目录下找到。

在doCreate方法的最后，`addSequentialTasks`向构造出的ddlJob中批量添加Task任务。作为最简单的Task组合接口，addSequentialTask以参数taskList中元素的下标顺序作为拓扑序构造Task之间的依赖关系。此方法揭示出在ddlJob中以DAG评接DDL Task的组合方式，DDL引擎中将根据DAG中的拓扑序调度DDL Task 。此外，在声明复杂依赖关系时还可使用`addTask`, `addTaskRelationShip`等方法单独声明Task及其依赖顺序。

## DDL与DML的同步

上一节中的`bringUpGsi`是online-schema-change流程的一个完整实现，定义于`com.alibaba.polardbx.executor.ddl.job.task.factory.GsiTaskFactory#addGlobalIndexTasks`方法中，我们以此为例介绍DDL与DML同步的几个关键组成部分。

```java
·    public static List<DdlTask> addGlobalIndexTasks(String schemaName,
                                                    String primaryTableName,
                                                    String indexName,
                                                    boolean stayAtDeleteOnly,
                                                    boolean stayAtWriteOnly,
                                                    boolean stayAtBackFill) {
        ....
        DdlTask writeOnlyTask = new GsiUpdateIndexStatusTask(
            schemaName,
            primaryTableName,
            indexName,
            IndexStatus.DELETE_ONLY,
            IndexStatus.WRITE_ONLY
        ).onExceptionTryRecoveryThenRollback();
        ....
        taskList.add(deleteOnlyTask);
        taskList.add(new TableSyncTask(schemaName, primaryTableName));
        ....
        taskList.add(writeOnlyTask);
        taskList.add(new TableSyncTask(schemaName, primaryTableName));
        ...
        taskList.add(new LogicalTableBackFillTask(schemaName, primaryTableName, indexName));
        ...
        taskList.add(writeReOrgTask);
        taskList.add(new TableSyncTask(schemaName, primaryTableName));
        taskList.add(publicTask);
        taskList.add(new TableSyncTask(schemaName, primaryTableName));
        return taskList;
    }

```

1.  元信息的多版本演化过程。索引的元信息经历从 CREATING -> DELETE_ONLY -> WRITE_ONLY -> WRITE_REORG -> PUBLIC的完整版本演化流程，其状态设计原理见online schema change一文，此状态设计过程保证系统中存在双版本元数据时不会出现不一致状态。
1.  DML在CN缓存的对应版本元信息下的重写。在元信息版本演化过程中，进入执行器的DML语句将会分派对应的rewriter根据索引状态生成实际的DML，从而满足write_only, update_only等元信息状态约束，并在数据回填时实现DML双写。
1.  MetaDB和CN缓存元信息之间的同步时序。sync操作将按一定的时序修改上述两部分元信息，一方面保证DML事务期间对应版本元信息的可用性，同时在相应版本DML事务结束后抢占MDL锁并invalidate原版本信息，另一方面通过提前加载新版本元信息，确保同一时刻其他DML事务总能获取可用的元信息。

## DML的重写

索引元信息中的状态定义见`com.alibaba.polardbx.gms.metadb.table.IndexStatus`. DDL更新该字段的同时，优化器为修改主表的DML语句在RBO中分配相应的gsiWriter,  以简单的Insert语句`insert into t1 values(1,2)`为例：

1.  Insert类型的语句在对应的RBO阶段`polardbx-optimizer/src/main/java/com/alibaba/polardbx/optimizer/core/planner/rule/OptimizeLogicalInsertRule.java`根据主表元信息中的gsiMeta在执行计划中关联到相应GSI的writer.

```java
//OptimizeLogicalInsertRule.java
private LogicalInsert handlePushdown(LogicalInsert origin, boolean deterministicPushdown, ExecutionContext ec){
      ...//other writers
        final List<InsertWriter> gsiInsertWriters = new ArrayList<>();
    IntStream.range(0, gsiMetas.size()).forEach(i -> {
        final TableMeta gsiMeta = gsiMetas.get(i);
        final RelOptTable gsiTable = catalog.getTableForMember(ImmutableList.of(schema, gsiMeta.getTableName()));
        final List<Integer> gsiValuePermute = gsiColumnMappings.get(i);
        final boolean isGsiBroadcast = TableTopologyUtil.isBroadcast(gsiMeta);
        final boolean isGsiSingle = TableTopologyUtil.isSingle(gsiMeta);
        //different write stragety for corresponding table type.
        gsiInsertWriters.add(WriterFactory
                             .createInsertOrReplaceWriter(newInsert, gsiTable, sourceRowType, gsiValuePermute, gsiMeta, gsiKeywords,
                                                          null, isReplace, isGsiBroadcast, isGsiSingle, isValueSource, ec));
        });
   ...
}
```

1.  writer在执行器的Handler中apply到物理执行计划中。其中`getInput`方法根据`gsiWriter`中包含的`indexStatus`状态信息决定是否生成当前DML对GSI的物理写操作，因此，最终DML仅在索引更新到WRITE_ONLY后开启对GSI的双写。

```java
//LogicalInsertWriter.java    
protected int executeInsert(LogicalInsert logicalInsert, ExecutionContext executionContext,
                                HandlerParams handlerParams) {
        ...
       final List<InsertWriter> gsiWriters = logicalInsert.getGsiInsertWriters();
       gsiWriters.stream()
                .map(gsiWriter -> gsiWriter.getInput(executionContext))
                .filter(w -> !w.isEmpty())
                .forEach(w -> {
                    writableGsiCount.incrementAndGet();
                    allPhyPlan.addAll(w);
                });

//IndexStatus.java
...
public static final EnumSet<IndexStatus> WRITABLE = EnumSet.of(WRITE_ONLY, WRITE_REORG, PUBLIC, DROP_WRITE_ONLY);

public boolean isWritable() {
   return WRITABLE.contains(this);
}
...
```

## 元信息同步时序

![](https://intranetproxy.alipay.com/skylark/lark/__puml/7ad1a62b930bf758933b2946c043109d.svg#lake_card_v2=eyJ0eXBlIjoicHVtbCIsImNvZGUiOiJAc3RhcnR1bWxcbmFjdG9yIHVzZXIgXG5cbnBhcnRpY2lwYW50IHVzZXIgXG5wYXJ0aWNpcGFudCBub2RlMCBcbnBhcnRpY2lwYW50IG5vZGUxLnN5bmNMaXN0ZW5lciBcbnBhcnRpY2lwYW50IG5vZGUxLm1kbC52MFxucGFydGljaXBhbnQgbm9kZTEubWRsLnYxXG5cbnBhcnRpY2lwYW50IG5vZGUxLnRocmVhZDFcbnBhcnRpY2lwYW50IG5vZGUxLnRocmVhZDJcblxudXNlciAtXFxcXCBub2RlMDogQUxURVIgVEFCTEUgdDEgQUREIEdMT0JBTCBJTkRFWFxcbiBnX2lfeSBvbiB0MSh5KVxuYWN0aXZhdGUgbm9kZTBcbmFjdGl2YXRlIG5vZGUxLnN5bmNMaXN0ZW5lclxuXG5ub2RlMCAtPiBub2RlMDogY3JlYXRlIGluZGV4IHRhYmxlIGFuZCBtZXRhIGRhdGFcblxubm9kZTAgLT4gbm9kZTEuc3luY0xpc3RlbmVyOiBzeW5jXG5cbnVzZXIgLT4gbm9kZTEudGhyZWFkMTogSU5TRVJUIElOVE8gdDFcbmFjdGl2YXRlIG5vZGUxLnRocmVhZDFcblxubm9kZTEudGhyZWFkMSAtPiBub2RlMS5tZGwudjA6IG1kbCh0MSkucmVhZExvY2soKVxuYWN0aXZhdGUgbm9kZTEubWRsLnYwXG5cbm5vZGUxLnN5bmNMaXN0ZW5lciAtPiBub2RlMS5zeW5jTGlzdGVuZXI6IDEuIGxvYWRTY2hlbWEodDEsIGMxLCBWMClcblxuXG5cblxuXG5ub2RlMS5zeW5jTGlzdGVuZXIgLT4gbm9kZTEuc3luY0xpc3RlbmVyOiBpcyBNREwgbmVlZGVkP1xubm90ZSBvdmVyIG5vZGUxLnN5bmNMaXN0ZW5lclxuICAgIEZvciBzdGF0ZSB0cmFuc2l0aW9uIGZyb20gd3JpdGVfb25seSB0byBwdWJsaWMsXG4gICAgTURMIGlzIG5vdCByZXF1aXJlZCBcbmVuZCBub3RlXG5ub2RlMS5zeW5jTGlzdGVuZXIgLS0-IG5vZGUwOiBzdWNjZXNzKGlmIE1ETCBpcyBub3QgcmVxdWlyZWQpXG5cbm5vZGUxLnN5bmNMaXN0ZW5lciAtPiBub2RlMS5zeW5jTGlzdGVuZXI6IGdldFNjaGVtYUZyb21TeXNUYWJsZSh0MSwgZ19pMSwgdjEpXG5cbm5vZGUxLnN5bmNMaXN0ZW5lciAtPiBub2RlMS5tZGwudjE6IGNyZWF0ZU1kbEZvck5ld1ZlcnNpb25NZXRhRGF0YVxuYWN0aXZhdGUgbm9kZTEubWRsLnYxXG5cbm5vZGUxLnN5bmNMaXN0ZW5lciAtPiBub2RlMS5tZGwudjA6IDIuIG1kbCh2MCkud3JpdGVMb2NrKClcblxuXG5ub2RlMS5tZGwudjAgLT4gbm9kZTEubWRsLnYwOiB3YWl0XG5cbnVzZXIgLT4gbm9kZTEudGhyZWFkMjogSU5TRVJUIElOVE8gdDFcbmFjdGl2YXRlIG5vZGUxLnRocmVhZDJcblxubm9kZTEudGhyZWFkMiAtPiBub2RlMS5tZGwudjE6IG1kbCh0MSkucmVhZExvY2soKVxuXG5ub2RlMS50aHJlYWQxIC0tPiBub2RlMS5tZGwudjA6IG1kbCh0MSkudW5sb2NrKClcblxuXG5ub2RlMS5tZGwudjAgLS0-IG5vZGUxLnN5bmNMaXN0ZW5lcjogbG9jayBhY3F1aXJlZFxuXG5cbm5vZGUxLnRocmVhZDEgLS0-IHVzZXI6IHN1Y2Nlc3NcbmRlYWN0aXZhdGUgbm9kZTEudGhyZWFkMVxuXG5cblxuXG5cbm5vZGUxLnN5bmNMaXN0ZW5lciAtPiBub2RlMS5zeW5jTGlzdGVuZXI6IDMuIGV4cGlyZVNjaGVtYU1hbmFnZXIodDEsIGdfaTEsIHYwKVxuXG5ub2RlMS5zeW5jTGlzdGVuZXIgLS0-IG5vZGUxLm1kbC52MDogbWRsKHYwKS51bmxvY2soKVxuXG5ub2RlMS5zeW5jTGlzdGVuZXIgLS0-IG5vZGUwOiBzdWNjZXNzOiBWMCBsb2FkZWRcblxuZGVhY3RpdmF0ZSBub2RlMS5tZGwudjBcblxubm9kZTAgLT4gbm9kZTA6IGhhbmRsZSByZXN0IG5vZGVzXG5cbm5vZGUxLnRocmVhZDIgLS0-IG5vZGUxLm1kbC52MTogbWRsKHYxKS51bmxvY2soKVxubm9kZTEudGhyZWFkMiAtLT4gdXNlcjogc3VjY2Vzc1xuZGVhY3RpdmF0ZSBub2RlMS50aHJlYWQyXG5cbm5vZGUwIC0tPiB1c2VyOiBzdWNjZXNzXG5kZWFjdGl2YXRlIG5vZGUwXG5AZW5kdW1sXG4iLCJ1cmwiOiJodHRwczovL2ludHJhbmV0cHJveHkuYWxpcGF5LmNvbS9za3lsYXJrL2xhcmsvX19wdW1sLzdhZDFhNjJiOTMwYmY3NTg5MzNiMjk0NmMwNDMxMDlkLnN2ZyIsImlkIjoiakZBMzkiLCJtYXJnaW4iOnsidG9wIjp0cnVlLCJib3R0b20iOnRydWV9LCJ3aWR0aE1vZGUiOiJjb250YWluIiwiY2FyZCI6ImRpYWdyYW0ifQ==)在通过sync同步MetaDB和内存元信息时，将依照以下步骤进行：

1.  `loadSchema`: 首先预加载MetaDB中的新版本元信息到内存中。加载完成后，内存中新增新版本元数据及其MDL锁，此后进入CN的DML(node1.thread2) 均会获取新版本元数据的MDL.
1.  `mdl(v0).writeLock`: 然后尝试获取旧版本元数据的MDL锁(node1.mdl.v0)，当获取锁成功时，旧版本事务即被排空。
1.  `expireSchemaManager(t1, g_i1, v0)`：消除旧版本元信息，将新版本元信息标记为旧版本元信息。

上述sync调用的响应者是CN集群中的所有节点（包括调用者本身），当所有节点均完成元信息版本切换时，sync调用即告成功，此时序保证了以下几点：

1.  任一时刻整个集群中至多存在两个版本元信息，并且至少有一个版本的元信息可加MDL读锁，因而进入CN的MDL语句永不会阻塞。
1.  旧版本元信息在loadSchema后即不再关联到MDL事务，可保证有限的状态切换时间。

其中1,2,3步均实现于`com.alibaba.polardbx.executor.gms.GmsTableMetaManager#tonewversion`方法。

```java
   public void tonewversion(String tableName, boolean preemptive, Long initWait, Long interval, TimeUnit timeUnit) {
        synchronized (OptimizerContext.getContext(schemaName)) {
            GmsTableMetaManager oldSchemaManager =
                (GmsTableMetaManager) OptimizerContext.getContext(schemaName).getLatestSchemaManager();
            TableMeta currentMeta = oldSchemaManager.getTableWithNull(tableName);

            long version = -1;


            ....//查询当前MetaDB中的元数据版本并将其赋值给vesion

            //1. loadSchema
            SchemaManager newSchemaManager =
                new GmsTableMetaManager(oldSchemaManager, tableName, rule);
            newSchemaManager.init();

            OptimizerContext.getContext(schemaName).setSchemaManager(newSchemaManager);

            //2. mdl(v0).writeLock
            final MdlContext context;
            if (preemptive) {
                context = MdlManager.addContext(schemaName, initWait, interval, timeUnit);
            } else {
                context = MdlManager.addContext(schemaName, false);
            }

            MdlTicket ticket = context.acquireLock(new MdlRequest(1L,
                    MdlKey
                        .getTableKeyWithLowerTableName(schemaName, currentMeta.getDigest()),
                    MdlType.MDL_EXCLUSIVE,
                    MdlDuration.MDL_TRANSACTION));

            //3. expireSchemaManager(t1, g_i1, v0)
            oldSchemaManager.expire();


            ....//失效使用旧版本元信息的PlanCache.

            context.releaseLock(1L, ticket);
        } 
    }
```

通过上述几个要素，DDL Job在定义时实现了DDL与DML的同步，保证了DML的online执行。

## 总结

本文主要解读了 PolarDB-X 中 CN 端的DDL Job定义相关的代码，以添加全局二级索引为例，对DDL Job定义和执行的整体流程进行了梳理，并着重阐述了DDL job定义中涉及online和crash safe特性的关键逻辑。对于DDL引擎中的DDL job执行流程，敬请期待下篇解读。



Reference:

