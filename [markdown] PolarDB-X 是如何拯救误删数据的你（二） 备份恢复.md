## 前言

上一篇文章中，我们介绍了 PolarDB-X 面向数据损坏，提供不同粒度的数据恢复能力，并针对行级数据误删场景，重点介绍了 PolarDB-X 的 SQL 闪回功能是如何帮助用户精确恢复被 DML 语句（DELETE，UPDATE）误删除的数据的，感兴趣的读者请参考《[PolarDB-X 如何拯救误删数据的你（一）](https://zhuanlan.zhihu.com/p/367137740)》。

本篇文章将针对 PolarDB-X 的备份恢复功能进行详细介绍。

## 面临的挑战

备份恢复是数据库的保障数据安全的必备能力。对于传统的单机数据库而言，备份恢复的技术已经比较完善，主要包括如下几种：

-   逻辑备份：数据库对象级备份，备份内容是表、索引、存储过程等数据库对象，如MySQL mysqldump、Oracle exp/imp。
-   物理备份：数据库文件级备份，备份内容是操作系统上数据库文件，如MySQL XtraBackup、Oracle RMAN。
-   快照备份：基于快照技术获取指定数据集合的一个完全可用拷贝，随后可以选择仅在本机上维护快照，或者对快照进行数据跨机备份，如文件系统Veritas File System、卷管理器Linux LVM、存储子系统NetApp NAS。

然而，对于 PolarDB-X 这样的分布式数据库，如何进行数据的备份与恢复，面临了诸多挑战。

### 任意时间点恢复（PITR）+ 全局一致性

任意时间点恢复（Point-in-time Recovery）是首先需要面临的挑战。PITR 指依赖备份集，将数据库恢复到过去的任意时间点（秒级）的能力。传统的数据库通常依赖单机的全量+增量的物理备份方式实现 PITR 能力，例如 MySQL 的XtraBackup + Binlog 等。

而对于分布式数据库，由于数据的读写涉及多个数据节点以及分布式事务的存在，在任意点恢复的过程中，除了需要保证单个节点的数据完整性外，也需要保证多个节点间的数据一致性。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/46860/1636115484727-184a080f-e563-47b4-89be-6b431d3cf81e.png)

上图通过经典的[转账测试](https://zhuanlan.zhihu.com/p/401355454)给出了数据一致性的例子。用户的账户余额表分布在两个数据节点中，某一时刻，通过分布式事务，账户C 向账户A转账了30元，账户D向账户C转账了20元。如果将数据节点恢复至该时刻，由于任意时间点恢复只能精确到秒级，因此恢复出的数据可能出现 A，C 账户完成了转账操作，而 B，D 账户还未完成的情况（如情况1），此时数据便出现了不一致，这对于用户是不可接受的。需要保证恢复出的数据，要么是转账前的状态，要么是转账完成后的状态。

### 业务无损

备份是数据库高频的运维操作，通过建议每天进行一次备份，保证数据安全。既然备份如此高频，那么数据备份的过程就要求对业务尽可能无损。如何在保证数据一致性的前提下，对数据库进行无损的备份，也是挑战之一。

### 扩展性

分布式数据库存储的数据量是远大于单机数据库的，通常在几十甚至上百TB。面对如此巨大的数据量，如何进行快速的数据备份与恢复？随着业务数据量的增长，如何保证备份恢复的速度也能线性增长，从而备份恢复的耗时相对稳定？都是我们需要解决的问题。

举个例子，当数据量 10TB 时，备份数据库需要1小时，如果数据规模增长到 100TB， 此时的备份时间仍需要保证在 1 小时左右，而不是增长到 10 小时。

## 方案解读

本节将介绍 PolarDB-X 的备份恢复方案，以及其是如何解决上述的挑战的。首先，我们回顾下单个数据节点是如何恢复至任意时间点（精确到秒级）的，然后再介绍 PolarDB-X 整体的备份恢复方案。

### 数据节点的数据恢复

当前 PolarDB-X 2.0 的数据节点（DN）采用的是在MySQL的基础上基于 X-Paxos 打造的分布式跨AZ高可用数据库，详情请参考：[PolarDB-X 存储架构之“基于Paxos的最佳生产实践”](https://zhuanlan.zhihu.com/p/315596644)。由于 DN 是基于 MySQL 打造的，因此其任意时间点的恢复方案与 MySQL 类似。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/46860/1636115533278-222210e3-5140-42fc-80e5-c04a58e1363b.png)

上图给出了单个数据节点的任意时间点备份恢复方式。首先通过 xtrabackup 对原实例进行定时的全量备份以及对binlog 的增量备份。当我们需要将节点恢复到某一时间点时（精确到秒级），首先找到最近的全量备份集，进行数据恢复，然后找到从全量备份集开始导恢复时间点之间的所有 binlog 文件，通过 MySQL 的 Crash Recovery的流程，将这部分binlog的event进行apply，即可将数据恢复到指定的时间点。

在上述的恢复流程中，头尾两个 binlog 文件需要单独处理。

-   **binlog-06**：由于全量备份的过程中，业务数据仍在正常写入，因此全量备份完成后，会记录备份时刻对应的binlog的pos信息，存储到备份集中，当应用binlog的时候，binlog-06 需要从全量备份集记录的pos开始apply，前面的部分需要丢弃
-   **binlog-12**：由于恢复的时间点是任意的，因此需要恢复的最后一条数据变更可能存在于binlog文件的任意位置。对于最后一个binlog文件，需要对其进行裁剪，将大于该恢复时间点的 binlog event 剔除。

### PolarDB-X 的数据恢复

回到 PolarDB-X 的数据恢复，依托单个 DN 的数据恢复能力，我们可以通过将当前实例下所有的DN 和 GMS 都恢复至同一时间点，做到任意时间点的恢复（PITR）。但是由于分布式事务的存在，这样是无法保证 DN 间的数据全局一致性的。

#### TSO In Binlog

PolarDB-X 的分布式事务为了提供外部一致性读，SI隔离级别的分布式事务能力，采用业内经典的TSO(TimeStampOracle)方案, 如下图所示。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/46860/1636115545814-866576c4-e207-41f7-9d5c-712035015d40.png)

在 TSO 分布式事务的实现中，对 DN 添加了 CTS（Commit Timestamp）扩展，为每个分布式事务，定义了两个序号，一个是事务启动时的序号 **snapshot_seq**（用于快照读），一个是事务提交的序号 **commit_seq**。这个序号由GMS统一分发，保证全局唯一且单调递增。详见：[PolarDB-X 分布式事务的实现（一）](https://zhuanlan.zhihu.com/p/338535541) 和 [PolarDB-X 全局时间戳服务的设计](https://zhuanlan.zhihu.com/p/360160666)。

除此之外，上面的事务序号也写入了 Binlog 中。在分布事务涉及到的每个DN节点的 Binlog 中，除了有常见的XA Start Event、XA Prepare Event和XA Commit Event之外，还会附带一个CTS Event来标识事务的开始和提交的时间，这个CTS Event保存的是一个具体的TSO时间戳。

#### 基于 TSO 的 Binlog 裁剪

既然 binlog 中记录了事务的 TSO，当我们需要进行全局一致的任意时间点恢复的时候，首先将需要恢复的时间点（例如：2021-07-25 16:14:21）转换成对应的 tso 时间戳，然后基于该 tso 对每个 DN 需要 apply 的 binlog 进行裁剪，**剔除那些事务提交序号（snapshot_seq） 大于该 tso 的事务 event 即可**，这样便能保证恢复出的数据不存在事务部分提交的不一致情况。

**恢复时间点转换成 TSO**

下图给出了 TSO 的格式，从图中可以看出TSO 时间戳是由物理时间戳 + 逻辑时间戳组合而成的64位数。考虑到 PITR 只需要精确到秒级，我们直接将需要恢复时间点（例如：2021-07-25 16:14:21）的时间戳左移 42 位，剩下的低22全部用0补齐，这样便能得到对应的 TSO 时间戳。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/46860/1636115558866-d68451ef-29ef-4b4e-9203-9be1332b24c3.png)

**Binlog 裁剪**

有了对应的 TSO 之后，接下来便是对 Binlog 进行裁剪。何时进行 Binlog 的裁剪，我们也考虑过多种方案：

-   方案1： 在 Binlog apply 的过程中进行裁剪
    -   实现原理：在 DN 启动的过程中，传入 TSO 时间戳，然后 DN 通过 Crash Recovery 流程 apply Binlong 的过程中对每个 event 的 TSO 进行判断是否需要剔除
    -   优点：纯内核侧实现，不依赖外部工具，无 binlog 空洞问题。
    -   缺点：对 MySQL Crash Recovery 的核心流程侵入较大，风险较高。同时由于涉及大量 TSO 的判断，影响 binlog apply 的性能

-   方案2： 在 DN apply binlog 前裁剪
    -   实现原理：在 DN apply binlog 前，通过外部的工具对 binlog 进行裁剪，剔除不必要的event，然后 DN 启动后直接 apply 裁剪好的 binlog 文件即可。
    -   优点: 对核心逻辑无侵入，binlog apply 性能无影响，稳定性好
    -   缺点：需要增加一个额外的工具，存在 binlog 的乱序及空洞问题

综合对比了上述两种方案，最终，我们采用了方案2。

**Binlog 乱序**

Binlog 的裁剪逻辑看似简单，只是 TSO 的大小比较和event的剔除。虽然事务获取TSO是有序的，但是由于节点间的网络延迟不同等因素的存在，实际提交过程中，落到每个 DN Binlog 中的 TSO 并不是严格有序的。下图便给出了一种 binlog event 的可能顺序。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/46860/1636115583634-e0e730f7-9462-4477-8b68-24a49e352f5e.png)

从上图中可以看出，事务2的提交TSO（100） 虽然小于事务3，但是在binlog中，这个event却是在事务3 提交的event 之后。这样的乱序问题就给我们的裁剪带了了问题：**何时终止裁剪过程？**

常规的裁剪逻辑通过是逐个往后遍历，一旦出现第一个大于比较TSO的值后，便可以终止流程。如下面的伪代码：

```C++
while(true) {
    if (event.tso > restore.tso) {
    	break;
    }
    move to next event
}
```

但是对于上述的binlog 的顺序，该方法便不再适用。例如当我们需要恢复的时间点的 TSO 是100，此时我们扫描到 C3的event的时候，发现该event的TSO 是 101，已经大于了100， 但是如果此时终止了裁剪逻辑，C2 这个该提交的event 便会被漏掉。

如何在这样的乱序情况下确定裁剪终止条件呢？这里考虑到binlog的乱序只是短时间范围内的，即不超过事务提交的超时时间。如果超出该时间，事务还未提交，该事务便会被 rollback，而不会 commit。利用该条件，我们引入一个新的变量 stop_tso:

```C++
stop_tso = （恢复时间点 + 事务超时时间（默认60s）+ delta）<< 42
```

其中：delta 是一个放大系数，默认我们采用了60s。

有了 stop_tso 之后，我们便能在出现第一条大于该值的event的时候终止裁剪流程。例如对上面的例子，恢复时间点的tso 是100， 但是 stop_tso 经过计算，是103，我们按照这个条件，便能对对应的binlog进行剔除，剔除后的效果如下图所示，P3，C3，C1 三个event都需要被剔除。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/46860/1636115677911-88b99a30-62c5-423e-b5bf-af5a5aacf1aa.png)

**悬挂事务处理**

仍然以上面的binlog event顺序为例，完成了binlog的裁剪后，当 DN 对 binlog apply 完成后，我们会发现虽然数据已经达到了一致的状态，但是细心的读者会发现，事务1因为 对应的 C1 event 被剔除了，但是 P1 event 仍然被apply了，虽然因为事务没提交，在恢复的数据中我们看不到对应的数据改动，但是这在我们的系统中属于悬挂事务，需要将这些悬挂的事务通过 XA Rollback进行回滚，要不便会影响后续事务的执行。

最终的效果应该如下图所示：

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/46860/1636115687495-fcfd3564-75cf-4413-8304-4762cf823318.png)

综上，基于 TSO 的 binlog 裁剪算法描述如下：

**输入**：待裁剪的binlog文件，restore_tso: 恢复时间点对应的tso时间戳，stop_tso: 停止扫描event的tso时间戳
**输出**：裁剪后的binlog 文件
**算法过程**：

1.  遍历 binlog 中的 event
1.  如果 event.tso >= stop_tso, 结束算法，否则前往步骤 3。
1.  如果 event.tso > restore_tso, 则跳过该 event，否则前往步骤 4。
1.  如果 event.tso <= restore_tso: 将 event 输出裁剪后的 binlog 文件。

a. 如果是 Prepare Event，将该event 写入集合S。
  b. 如果是 Commit Event 或者 Rollback Event，在集合 S 中删除对应的 Prepare Event。
5. 遍历结束后删除集合 S 中的Prepare Event，进行事务 Rollback

#### 业务影响 & 扩展性

读到这里，PolarDB-X 保证全局一致的备份恢复方案基本就介绍完了，那么这套方案能否做到业务无损和可扩展呢？

由于我们的备份操作主要在 DN 节点上执行，而 DN 是采用了 X-Paxos 协议的 MySQL 三节点，因此我们只需要将备份操作放到 Follower 节点上执行，由于 Follower 节点不承接业务的流量，因此备份的过程对业务的影响基本可以忽略。

对于扩展性问题，由于备份恢复操作都是每个 DN 单独进行的，DN 间并不需要进行很重的协同操作（主要就是恢复时间点的下发与同步，该操作十分轻量），因此当我们的实例从10 TB（10 个DN） 扩展到 100 TB（100个DN）的时候，整体的备份恢复时间仍然接近单个 DN 的备份恢复时间，不会随着数据量的增加而迅速增长。

## 总结与展望

本文主要针对 PolarDB-X 的备份恢复方案，详细介绍了其是如何保证数据的全局一致性以及如何做到业务无损且可扩展的。

当然，现有的这套备份恢复方案也存在一些未解问题，比如备份期间不能进行扩缩容、仅支持同构恢复等，这些将是我们接下来的优化方向。



Reference:

