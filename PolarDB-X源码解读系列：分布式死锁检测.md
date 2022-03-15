本文将主要解读 PolarDB-X 中分布式死锁检测功能的源码。阅读前建议先了解我们分布式死锁检测原理的相关文章：

-   [PolarDB-X 分布式MDL死锁检测](https://zhuanlan.zhihu.com/p/438968674)

# 死锁检测任务的生命周期

死锁检测功能属于事务模块的功能，死锁检测任务则挂载在事务管理器 [TransactionManager](https://github.com/ApsaraDB/galaxysql/blob/main/polardbx-transaction/src/main/java/com/alibaba/polardbx/transaction/TransactionManager.java) 中。在 TransactionManager 初始化时（参考 [PolarDB-X CN 启动流程](https://zhuanlan.zhihu.com/p/443608658)，代码入口在 [MatrixConfigHolder#doInit](https://github.com/ApsaraDB/galaxysql/blob/main/polardbx-server/src/main/java/com/alibaba/polardbx/matrix/config/MatrixConfigHolder.java) 中调用了对应 transactionManager#init 方法），TransactionManager 会初始化一个死锁检测的定时任务，该任务每个计算节点（CN）都有且只有一个，每隔一定时间进行一次死锁检测（默认是 1 秒）。

接下来看一下这个死锁检测的任务里面具体做了什么事情。

# 代码主体逻辑：DeadlockDetectionTask

死锁检测任务的代码主要在 [DeadlockDetectionTask](https://github.com/ApsaraDB/galaxysql/blob/main/polardbx-transaction/src/main/java/com/alibaba/polardbx/transaction/async/DeadlockDetectionTask.java) 里，任务每次被调度时，执行的代码入口是其中的 run 方法。

该方法首先会判断一下当前 CN 是否为 leader，只有 leader 节点才会执行死锁检测任务。

```java
if (!hasLeadership()) {
    return;
}
```

一次死锁检测主要涉及三个步骤。

第一步，先获取所有分布式事务的信息。

```java
// Get all global transaction information
final TrxLookupSet lookupSet = fetchTransInfo();
```

这一步需要 sync 到所有 CN 节点，以获取所有分布式事务的信息，fetchTransInfo 方法里主要是调用了 [FetchTransForDeadlockDetectionSyncAction#sync](https://github.com/ApsaraDB/galaxysql/blob/main/polardbx-transaction/src/main/java/com/alibaba/polardbx/transaction/sync/FetchTransForDeadlockDetectionSyncAction.java) 这个 sync 的方法。为了避免返回过多的事务信息，这里面只会返回从开始到现在大于 1 秒的分布式事务。

```java
final long beforeTimeMillis = System.currentTimeMillis() - 1000L;
final long beforeTxid = IdGenerator.assembleId(beforeTimeMillis, 0, 0);

for (ITransaction tran : transactions) {
    if (!tran.isDistributed()) {
    	continue;
    }
    // Do deadlock detection only for transactions that take longer than 1s.
    if (tran.getId() >= beforeTxid) {
    	continue;
    }
    // Get information from this tran.
    ......
}
```

返回的 TrxLookupSet 记录了所有事务的信息，主要包括事务 id，事务所在前端连接的 id，事务涉及的分片，分片上分支事务（MySQL 上的事务）的连接 id 等。以上信息均会用在后续的死锁检测中。

第二步，获取每个数据节点（DN）的锁等待信息，并结合上一步的分布式事务信息，更新全局的事务等待关系图。

首先会获取所有 DN 的数据源。下面 Map 返回的数据中，key 是 DN 的 id，value 是一个列表，存放了所有 schema 对应的数据源。我们可以使用任意一个数据源来访问 DN，同时我们也会使用到这些数据源中存放的物理分片名和分支事务的连接 id 来确定对应的分布式事务。注意，只使用分支事务的连接 id 无法确定对应的分布式事务，因为不同的 DN 上能同时存在相同的连接 id。

```java
// Get all group data sources, and group by DN's ID (host:port)
final Map<String, List<TGroupDataSource>> instId2GroupList = ExecUtils.getInstId2GroupList(allSchemas);
```

然后我们会对每个 DN，获取物理分片上的锁和分支事务信息，结合上一步获取的 TrxLookupSet 中的分布式事务信息，更新一个全局的事务等待关系图（代码中的 DiGraph）。DiGraph 是一个全局事务等待关系图，图中每个点是一个事务，每条有向边表示事务的等待关系。

```java
final DiGraph<TrxLookupSet.Transaction> graph = new DiGraph<>();
for (List<TGroupDataSource> groupDataSources : instId2GroupList.values()) {
    if (CollectionUtils.isNotEmpty(groupDataSources)) {
    	// Since all data sources are in the same DN, any data source is ok.
    	final TGroupDataSource groupDataSource = groupDataSources.get(0);

    	// Get all group names in this DN.
    	final Set<String> groupNames =
    		groupDataSources.stream().map(TGroupDataSource::getDbGroupKey).collect(Collectors.toSet());

    	// Fetch lock-wait information for this DN,
    	// and update the lookup set and the graph with the information.
    	fetchLockWaits(groupDataSource, groupNames, lookupSet, graph);
    }
}
```

fetchLockWaits 方法主要就是查询 DN 上 information_schema.innodb_locks/innodb_trx/innodb_lock_waits 这三个视图，来获取具体的行锁信息。获取的内容主要包括锁等待的分支事务的连接 id，以及一些最后输出到死锁日志里的信息。然后，根据这个分支事务的连接 id，查到对应的分布式事务，将这个分支事务的等待关系转化为分布式事务的等待关系，加到等待图中。

最后一步，检测图中是否存在环，若存在，则回滚环中的某个事务。

```java
graph.detect().ifPresent((cycle) -> {
    // 若检测到环，先保留一份死锁日志
    DeadlockParser.parseGlobalDeadlock(cycle);
    // 然后选择一个事务回滚掉，目前默认回滚环的第一个事务
    killByFrontendConnId(cycle.get(0));
});
```

其中，`graph.detect()` 为检测环的算法，具体实现在 [DiGraph](https://github.com/ApsaraDB/galaxysql/blob/main/polardbx-transaction/src/main/java/com/alibaba/polardbx/transaction/utils/DiGraph.java) 部分的代码中。简单来说，就是在有向图上进行深度优先搜索，检测是否存在环路。此外，保留的死锁日志可以使用 SHOW GLOBAL DEADLOCKS 查看。

最后，为了解决死锁，会选择回滚掉环中的一个事务。目前默认是回滚环里的第一个事务，相当于随机回滚一个事务。

关于 MDL 死锁检测的代码与此类似，主体部分在 [MdlDeadlockDetectionTask](https://github.com/ApsaraDB/galaxysql/blob/main/polardbx-transaction/src/main/java/com/alibaba/polardbx/transaction/async/MdlDeadlockDetectionTask.java)。

# 其他细节

**1. 第一步的 sync 机制是如何实现的？**
简单来说，sync 的语义就是要让所有 CN 节点执行同一个 action，并得到 action 的结果。发起 sync 的节点会将该 action 对象序列化后发送到所有 CN，CN 反序列化后执行这个对象的 sync 方法，并发得到的结果返回。以上行为的逻辑主要实现在 [ClusterSyncManager#doSync](https://github.com/ApsaraDB/galaxysql/blob/main/polardbx-server/src/main/java/com/alibaba/polardbx/ClusterSyncManager.java) 方法里。阅读该部分代码，我们也能知道，给其他 CN 发送 sync 请求其实就是执行了一个 `SYNC schema_name serializedAction` 的 SQL 语句，返回的结果也是和普通查询返回的结果类似。

**2. 第三步中，具体是如何回滚事务的？**
发生死锁时，待回滚的事务当前执行的语句正卡在某个锁等待上。因此，死锁检测任务会发起一个 kill query 的 sync 请求，先把这条语句 kill 掉，并把 kill query 的错误码设置为 _ERR_TRANS_DEADLOCK_。kill query 的入口在 [KillSyncAction](https://github.com/ApsaraDB/galaxysql/blob/main/polardbx-server/src/main/java/com/alibaba/polardbx/server/response/KillSyncAction.java)，最后实际运行 kill query 逻辑的代码为  [ServerConnection.CancelQueryTask](https://github.com/ApsaraDB/galaxysql/blob/main/polardbx-server/src/main/java/com/alibaba/polardbx/server/ServerConnection.java) 中的 `doCancel` 方法。

```java
private void doCancel() throws SQLException {
    // 这个 futureCancelErrorCode 用在后面的错误判断中，
    // 死锁导致的 kill，错误码都是 ERR_TRANS_DEADLOCK
    futureCancelErrorCode = this.errorCode;

    // kill 掉所有物理连接上正在运行的 SQL
    if (conn != null) {
    	conn.kill();
    }

    // 这里这个 f 是正在执行逻辑 SQL 的任务
    Future f = executingFuture;
    if (f != null) {
    	f.cancel(true);
    }
}
```

该方法实际上会对每个物理连接（CN 和 DN 的连接）调用 kill query 的逻辑，把该逻辑语句对应的所有物理语句都 kill 掉，其中正在等锁的物理语句就会被 kill 掉，最后会中断正在执行这个逻辑语句的线程。

被中断的线程会在 [ServerConnection](https://github.com/ApsaraDB/galaxysql/blob/main/polardbx-server/src/main/java/com/alibaba/polardbx/server/ServerConnection.java) 中的 `handleError` 方法里处理异常，发现错误码是 *ERR_TRANS_DEADLOCK* 时，就会将当前事务回滚掉，并给客户端发送 *Deadlock found when trying to get lock; try restarting transaction* 的错误提示。

```java
// Handle deadlock error.
if (isDeadLockException(t)) {
    // Prevent this transaction from committing.
    this.conn.getTrx().setCrucialError(ERR_TRANS_DEADLOCK);

    // Rollback this trx.
    try {
    	innerRollback();
    } catch (SQLException exception) {
    	logger.warn("rollback failed when deadlock found", exception);
    }
}
```

# 小结

本文简单介绍了 PolarDB-X 分布式死锁检测功能的源码，感兴趣的同学可以结合源码解读，在 [DeadlockDetectionTask#run](https://github.com/ApsaraDB/galaxysql/blob/main/polardbx-transaction/src/main/java/com/alibaba/polardbx/transaction/async/DeadlockDetectionTask.java) 方法打一个断点，观察每一步执行的结果，可以更容易理解这一功能的实现。



Reference:

