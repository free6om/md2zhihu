# 背景

一般而言，在数据库中，死锁指的是事务间因互相持有了对方需要的锁而导致事务无法继续进行下去的现象。在 PolarDB-X 中，死锁可以分成三种类型：单机死锁（即单 DN 上的死锁），分布式死锁，以及[分布式 MDL 死锁](https://zhuanlan.zhihu.com/p/438968674)（即分布式元数据锁的死锁）。本文将主要介绍 PolarDB-X 如何检测分布式死锁，以及当死锁出现后，用户可以怎样分析死锁。

## 分布式事务和分支事务

首先，我们简要介绍一下 PolarDB-X 的事务系统。在 PolarDB-X 中，一张逻辑表可能被划分到不同的物理分片上。PolarDB-X 的存储节点（DN）是基于 MySQL 开发的 XDB，因此，每个物理分片可以简单理解为 MySQL 中的一个物理库。在执行分布式事务时，PolarDB-X 会在每个物理分片上开启一个分支事务。分支事务则可以简单理解为 MySQL 中的事务。对于 DN 上的死锁，MySQL 已经具备了检测的能力，基本原理是构造锁等待的有向图，然后检测其中是否存在环，在检测出环后自动回滚某个事务，以打破环而解决死锁。

但是，对于分布式死锁，由于触发死锁的分支事务位于不同的分片上，MySQL 无法检测出这种类型的死锁。以下图为例，分布式事务 A 的分支事务 A1 正在等待分布式事务 B 的分支事务 B1 所持有的锁；而 B 的分支事务 B2 正在等待 A 的分支事务 A2 所持有的锁。这是一个典型的死锁场景，事务 A 和 B 在互相等待对方持有的锁。但对于 MySQL 来说，这里只存在两个锁等待关系（A1->B1, B2->A2），并不形成锁等待的环。更常见的是，A1 和 B1 所在的分片 1，与 A2 和 B2 所在的分片 2，位于不同的 DN 上，因此在每个 DN 上只能看到一个锁等待关系。

![分布式死锁](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/315529/1639133313943-003a0c5b-443a-4171-9cac-3a98a830a8d0.png)

# PolarDB-X 如何检测分布式死锁

在出现死锁时，如果没有人工干预，参与死锁的事务将永远无法继续进行下去，直至锁超时等错误发生。因此，死锁检测是一个分布式数据库必须具备的能力。对于 CockroachDB，其每个节点都会维护一个内存中的 lock table，用于记录锁的分配情况，其中也会维护锁的等待关系信息。同时，该锁等待关系会定时地推送到每个 Range 的raft leader 节点，由该 leader 节点维护并更新事务的等待关系信息，并检测其中是否存在相互等待的环，检测出环就意味着存在死锁。对于 TiDB，其在特定 region leader 所在的 TiKV 实例上维护了一个全局的锁等待关系图。在事务需要加锁但被阻塞时，会在图中增加一条边，如果新增边后会形成环，就意味着会产生死锁。

对于 PolarDB-X，其死锁检测的实现原理与 CockroachDB 的做法类似。首先，锁的分配情况分布在每个 DN 上，而分布式事务的运行状态则保存在 CN 上。因此，CN 的 leader 节点会定时地从每个 DN 收集锁等待关系，并从每个 CN 收集事务的信息。最后根据这些信息构造出事务的等待关系图，并检测图中是否存在环，在检测出环后 kill 掉环中的某个事务，以此来解决死锁问题。

在具体实现上，在单个 DN 上的 information_schema 中提供了 innodb_trx 和 innodb_locks 等视图，用于查看实时的锁信息和（分支）事务信息，通过 PolarDB-X 的 CN 节点可以方便地从中收集每个 DN 上锁的信息并构建出事务等待关系图。

# 使用“锁分析”功能

在出现死锁后，PolarDB-X 会 kill 掉某个事务，来解除死锁，此时客户端会收到相应的死锁报错。除了报错信息，我们还有什么途径能获取更多死锁相关的信息呢？

为了方便用户分析死锁的场景，PolarDB-X 控制台增加了“锁分析”的功能，具体入口可以在控制台左侧导航栏处找到。如下图所示，我们可以点击“发起诊断”来分析在 PolarDB-X 中最近发生的一次单机死锁/分布式死锁/分布式 MDL 死锁，可以点击“查看详情”看到死锁的分析结果。
![锁分析页面](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/315529/1636616819586-351cf63b-39e0-4a4b-9885-9480654778e5.png)

## 分布式死锁

以分布式死锁为例，点开详情页后，如下图所示，显示死锁类型为 GLOBAL。注意，图中的事务 SQL 是触发死锁时的逻辑 SQL，即用户真正输入的 SQL；而分支事务 SQL 则是在物理分片上实际执行的物理 SQL。值得一提的是，这里事务的编号体现了事务间的等待关系，即事务 i 正在等待事务 i+1 所持有的锁；最后一个事务正在等待第一个事务持有的锁。

根据详情页信息，查看等待锁和事务 SQL 的信息，发现事务 1 的分支事务 1 正在等待物理分片 db1_000000 上物理表 tb0_giuu 的 id=0 的行锁，而这个锁正在被事务 2 的分支事务 1 所持有；同时，事务 2 的分支事务 2 正在等待物理分片 db1_000001 上物理表 tb0_giuu 的 id=1 的行锁，而这个锁正在被事务 1 的分支事务 2 所持有。因此，从总体上看，事务 1 和 事务 2 构成了循环的锁等待，从而导致了死锁，最终回滚了事务 1 来解决这个死锁。

![分布式死锁详情页面](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/315529/1636618795719-409ab290-bda1-44cb-a221-43f2519dafb5.png)

对于单机死锁，详情页呈现的内容和分布式死锁类似，这里不再举例赘述。

## 分布式 MDL 死锁

对于分布式 MDL 死锁，除了事务，还会涉及 DDL 语句。如下图所示，点开详情页后显示死锁类型为 MDL，图中事务和 DDL 的编号体现了事务/DDL 间对于 MDL 的等待关系。

![分布式 MDL 死锁详情页面](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/315529/1636683908398-ae426e8a-dd36-42a5-92c6-ee105e2bab16.png)

由于 MDL 死锁涉及信息更繁杂，我们可以根据图中的信息，来还原出当初的死锁场景，更直观地分析死锁原因：

![MDL 死锁](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/315529/1645504773876-2977b1da-68ac-4430-975b-7506ff50c587.png)

DDL 语句想要获取 EXCLUSIVE (X) 锁时，需要等待该表上所有 SHARED_WRITE (SW) 锁被释放，同时，会阻塞后续所有 SW 锁的获取。通过上图中可以看到，在每个分片上（灰色和橙色分别表示分片 0 和分片 1），都没有形成锁等待的环。但总体上看，形成了“事务 1 -> DDL 2 -> 事务 3 -> DDL 4 -> 事务 1”这样循环的锁等待，从而导致了 MDL 死锁。结合详情页信息可知，最终回滚了事务 1 来解决这个死锁。

关于 MDL 之间的阻塞关系，由于内容比较长，不便在此处给出，感兴趣的可以进一步参考文末的附录。

# SHOW 
[LOCAL | GLOBAL]
 DEADLOCKS

其实，上述锁分析的功能，是对 PolarDB-X 的 SHOW [LOCAL | GLOBAL] DEADLOCKS 指令的解析。当出现死锁时，我们也可以直接使用该指令看到最近的一次死锁日志。此外，我们也把该死锁日志保留在了“锁分析”的结果中，点击死锁详情页的“查看死锁日志”就能看到了。因此，我们可以使用控制台的“锁分析”来看到死锁的主要信息，也可以进一步查看完整的死锁日志来获取更多的额外死锁信息。

# 总结

死锁会使得资源利用率降低，事务并发性能受损。更严重的是，MDL 死锁得不到解决，被锁的表上的事务甚至会被阻塞一年。在本文中，我们介绍了 PolarDB-X 如何检测和分析死锁。在检测出死锁后，PolarDB-X 会 kill 掉参与死锁的某个事务，从而解决死锁。这之后，用户可以使用“锁分析”功能自主分析死锁形成原因，以便在业务上排查和避免死锁。

# References

[PolarDB-X 分布式MDL死锁检测](https://zhuanlan.zhihu.com/p/438968674)
[cockroachDB 的事务系统与死锁检测](https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html#txnwaitqueue)
[TiDB 的悲观锁实现原理与死锁检测](https://asktug.com/t/topic/33550)
MySQL 8.0.26 源码

# 附录

可能出现在“锁分析”详情页的 MDL 锁类型有以下几种：

-   INTENTION_EXCLUSIVE (IX) ：一种范围意向锁，获取某个对象的 EXCLUSIVE (X) 锁前，都会获取某个 IX 锁。例如，要想获取某个表的 X 锁，会首先获取这个表所在库的 IX 锁，再尝试获取这个表的 X 锁。
-   SHARED (S)：当我们只对读取某个对象的元数据感兴趣，而不会读写这个对象本身的数据时，我们会获取这个对象的 S 锁。例如，在给某个用户授权某个表时，会获取一下这个表的 S 锁，以确定这个表是存在的。
-   SHARED_HIGH_PRIO (SH)：S 锁的加强版，当要获取这种锁时，可以无视等待队列直接获取。绝大部分的 SHOW 指令都会获取要 SHOW 的对象的 SH 锁。例如，当某个 ALTER TABLE 正在执行时，SHOW TABLES 是不会被阻塞的。
-   SHARED_READ (SR): 在 S 锁的基础上，不仅允许读写对象的元数据，还允许去读这个对象的数据。例如，普通的 SELECT 语句会获取表的 S 锁。
-   SHARED_WRITE (SW): 在 SR 锁的基础上，还允许去修改这个对象的数据。例如，几乎所有的 DML 语句都会获取表的 SW 锁；SELECT ... FOR UPDATE 也会获取表的 SW 锁。
-   MDL_SHARED_WRITE_LOW_PRIO (SWLP): 低优先级的 SW 锁。一些低优先级的 DML 语句会获取这个锁，例如 UPDATE LOW_PRIORITY 语句。
-   SHARED_UPGRADABLE (SU): 当获取了某个对象的 SU 锁时，仍然允许其他线程对这个对象进行读写；持有 SU 的线程则只可以读该对象的元数据和其本身的数据，不可以修改。例如，几乎所有的 ALTER TABLE 都会首先获取 TABLE 的 SU 锁。SU 锁会升级为 SHARED_NO_WRITE (SNW)，SHARED_NO_READ_WRITE (SNRW)，或是 X 锁，当升级为 XNRW 和 X 锁时，就可以修改这个对象的元数据和数据了。
-   SHARED_READ_ONLY (SRO): 当获取了某个对象的 SRO 锁时，仍然允许其他线程对这个对象进行读，但不允许写；持有 SRO 的线程则只可以读该对象的元数据和其本身的数据。使用 LOCK TABLES READ 时会获取 SRO 锁。
-   SHARED_NO_WRITE (SNW): 当获取了某个对象的 SNW 锁时，仍然允许其他线程对这个对象进行读，但不允许写；持有 SNW 的线程则只可以读该对象的元数据和其本身的数据，不可以修改。例如，ALTER TABLE 涉及从一个表拷贝数据到另一个表时，当获取了这些表的 SNW 时，就可以开始 SELECT 这些表，但还不可以 UPDATE。SNW 锁可以升级为 X 锁。
-   SHARED_NO_READ_WRITE (SNRW): 当获取了某个对象的 SNRW 锁时，只允许其他线程读这个对象的元数据，不允许写，也不允许读写对象的数据；持有 SNRW 的线程则可以读该对象的元数据，以及修改对象本身的数据。例如，使用 LOCK TABLES WRITE 时会获取表的 SNRW 锁。SNRW 锁可以升级为 X 锁。
-   EXCLUSIVE (X): 互斥锁，一旦获取某个对象的 X 锁，不允许其他线程读写该对象的元数据和数据。同时，获取 X 锁的线程可以修改对象的元数据和数据。例如，CREATE/DROP/RENAME TABLE 时会获取表的 X 锁。一些 DDL 的特定阶段也会获取 X 锁。

最后，结合下面的锁的等待-阻塞关系表，我们就能通过 MDL 锁类型来推断出具体死锁的根本原因了。

首先是范围锁，下表表示对于某个范围，已经持有了某种 MDL 时，获取另一种 MDL 会不会被阻塞，每一行表示请求的锁类型，每一列表示已持有的锁类型。其中，“-” 表示会被阻塞。可以看到，当想获取 X 锁时，需要等待所有锁被释放；在获取之后，其他语句就不能获取任何锁。

<table>
<tr class="header">
<th></th>
<th>IX</th>
<th>S</th>
<th>X</th>
</tr>
<tr class="odd">
<td>IX</td>
<td>+</td>
<td>-</td>
<td>-</td>
</tr>
<tr class="even">
<td>S</td>
<td>-</td>
<td>+</td>
<td>-</td>
</tr>
<tr class="odd">
<td>X</td>
<td>-</td>
<td>-</td>
<td>-</td>
</tr>
</table>

而下表则表示，当 MDL 的等待队列中已存在某种 MDL 时，获取另一种 MDL 会不会被阻塞，每一行表示请求的锁类型，每一列表示队列中正在等待的锁类型。其中，“-” 表示会被阻塞。可以看到，当队列中已有 X 锁在等待时，获取其他非 X 锁类型的锁都会被阻塞，即会被排在 X 锁的后面。

<table>
<tr class="header">
<th></th>
<th>IX</th>
<th>S</th>
<th>X</th>
</tr>
<tr class="odd">
<td>IX</td>
<td>+</td>
<td>-</td>
<td>-</td>
</tr>
<tr class="even">
<td>S</td>
<td>+</td>
<td>+</td>
<td>-</td>
</tr>
<tr class="odd">
<td>X</td>
<td>+</td>
<td>+</td>
<td>+</td>
</tr>
</table>

从图中也能看出，这三种范围锁的优先级顺序为：X > S > IX.

其次是对象上的 MDL 锁等待-阻塞关系表，下表表示对于某个对象，已经持有了某种 MDL 时，获取另一种 MDL 会不会被阻塞。

<table>
<tr class="header">
<th></th>
<th>S</th>
<th>SH</th>
<th>SR</th>
<th>SW</th>
<th>SWLP</th>
<th>SU</th>
<th>SRO</th>
<th>SNW</th>
<th>SNRW</th>
<th>X</th>
</tr>
<tr class="odd">
<td>S</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
</tr>
<tr class="even">
<td>SH</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
</tr>
<tr class="odd">
<td>SR</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
<td>-</td>
</tr>
<tr class="even">
<td>SW</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
</tr>
<tr class="odd">
<td>SWLP</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
</tr>
<tr class="even">
<td>SU</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
<td>+</td>
<td>-</td>
<td>-</td>
<td>-</td>
</tr>
<tr class="odd">
<td>SRO</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
<td>-</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
<td>-</td>
</tr>
<tr class="even">
<td>SNW</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>+</td>
<td>-</td>
<td>-</td>
<td>-</td>
</tr>
<tr class="odd">
<td>SNRW</td>
<td>+</td>
<td>+</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
</tr>
<tr class="even">
<td>X</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
</tr>
</table>

而下表表示对于某个对象，其 MDL 等待队列已存在某种 MDL 时，获取另一种 MDL 会不会被阻塞。

<table>
<tr class="header">
<th></th>
<th>S</th>
<th>SH</th>
<th>SR</th>
<th>SW</th>
<th>SWLP</th>
<th>SU</th>
<th>SRO</th>
<th>SNW</th>
<th>SNRW</th>
<th>X</th>
</tr>
<tr class="odd">
<td>S</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
</tr>
<tr class="even">
<td>SH</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
</tr>
<tr class="odd">
<td>SR</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
<td>-</td>
</tr>
<tr class="even">
<td>SW</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
<td>-</td>
<td>-</td>
</tr>
<tr class="odd">
<td>SWLP</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
</tr>
<tr class="even">
<td>SU</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
</tr>
<tr class="odd">
<td>SRO</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
<td>-</td>
</tr>
<tr class="even">
<td>SNW</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
</tr>
<tr class="odd">
<td>SNRW</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>+</td>
<td>-</td>
</tr>
<tr class="even">
<td>X</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
<td>-</td>
</tr>
</table>

举个例子：当某对象上已经上了 X 锁，则此时获取任何锁都会阻塞，都需要等待这个已经被授予的 X 锁。当某对象的 MDL 等待队列中已经有一个 X 锁，则此时获取任何除了 SH 锁和 X 锁的请求都会被阻塞，且该阻塞的请求需要等待这个 X 锁（换句话说，被阻塞的请求都排在这个 X 锁的后面，直到这个 X 锁被授予且被释放，才可能获取对应的锁）。



Reference:

