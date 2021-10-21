
## 背景

在之前的两篇文章中，我们分别介绍了 PolarDB-X [基于 TSO 的分布式事务](https://zhuanlan.zhihu.com/p/338535541) 以及 [对私有化 InnoDB 做的改造](https://zhuanlan.zhihu.com/p/355413022)。在 PolarDB-X 2.0 中，我们最新实现了一个异步提交（async commit）的特性。通过这个最新的优化，PolarDB-X **对于任意分布式事务的提交，可以做到跟单机事务相似的延迟**，对于一些延迟敏感的场景，用户也可以放心地使用分布式事务而不用为提交延迟精心优化。在这篇文章中，我将会重点介绍这个优化的设计和实现细节。

这个优化参考了 [CockroachDB Parallel Commits](https://www.cockroachlabs.com/blog/parallel-commits/) 的实现，我们也引入了一些额外的设计保障了 [Linearizability](https://zhuanlan.zhihu.com/p/360690247)。

**方案回顾**

首先我们再简单回顾一下优化前的 TSO 事务提交流程。TSO 事务是我们基于 XA 事务的改造，基本流程还是以两阶段提交为核心的：

![async_commit_before.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/295107/1618551086385-30bb38b8-44fb-4508-a5cf-971bfd6dd887.png)

1.  在 Prepare 阶段中，CN 先并行地发送 XA PREPARE 将事务的所有分支置为 Prepared 状态，在这个状态下，DN 可以阻塞涉及该事务修改的读取，以正确地可重复读的隔离级别（参考 [对私有化 InnoDB 做的改造](https://zhuanlan.zhihu.com/p/355413022)）。
1.  进入提交阶段后，CN 先访问 TSO 获取一个 COMMIT_TS，然后在 Primary 的 DN 上记录事务日志并执行 XA COMMIT ，直到这里就到了 **COMMIT POINT** 了，可以返回用户提交成功。
1.  异步对 SECONDARY 分支执行 XA COMMIT，解除读阻塞。

**什么是事务日志？**

COMMIT POINT 的核心是事务日志的持久化，让我们先看一下什么是事务日志。事务日志是每个 DN 上的一张系统表 `GLOBAL_TX_LOG`，用于记录每个事务的提交状态：

```sql
CREATE TABLE `GLOBAL_TX_LOG` (
  `TRX_ID` BIGINT PRIMARY KEY,
  `COMMIT_TS` BIGINT DEFAULT NULL,
  `STATE` ENUM('COMMITTED', 'ABORTED') NOT NULL
  # ...
);

# commit phase
INSERT INTO `GLOBAL_TX_LOG` VALUES (123, 100, "COMMITTED");
```

COMMIT POINT 就是在完成所有节点的 PREPARE 后，往事务日志表 INSERT 一条记录。事实上，事务日志就是为了解决两阶段协议中最经典的一个问题：协调者（CN）发生异常可能导致参与者（DN）状态不一致，通过事务日志的记录，协调者的状态被持久化了下来，这样即使协调者发生异常，也可以由新的协调者主导，通过读取事务日志获取事务状态，决定是继续提交事务还是回滚事务。

为了保证正确地实现分库之间的一致性，不同 DN 上的事务分支也必须使用同一个 COMMIT_TS，因此获取 COMMIT_TS 必须发生在记录事务日志之前，才能被持久化在事务日志中。

**为什么需要事务日志？**

我们梳理一下以后就不难发现，在完成 prepare phase 之后，所有数据都已经在 DN 上持久化了，但我们仍然必须等到 COMMIT POINT 才能返回用户提交成功，这导致了比较大的提交延迟。那么如果我们在 commit phase 之前直接**返回用户提交成功**，会发生什么问题呢？

假如 CN 发生异常，旧的 CN 发生意外不可用，新的 CN 发现 DN 上有处于 PREPARED 状态的事务，但没有在 PRIMARY 上找到事务日志，此时是无法判断是应该提交还是回滚 —— 新的 CN 无法区分两种处理方案冲突的情况：

1.  之前的事务处于 prepare phase，部分事务分支还没收到 XA PREPARE，部分事务分支处于 PREPARED 状态。此时必须回滚事务，因为部分分支已经不可恢复。

1.  之前的事务已经完成了 prepare phase，所有事务分支都收到了 XA PREPARE，但还没记录事务日志。此时必须提交事务，因为之前的 CN 可能**已经返回用户提交成功**。

**优化方案：两阶段的事务日志**

为了区分上文提到的两种情况，如果新的 CN 可以找到事务的所有分支，只需要去所有分支上查询一下是否都有 PREPARED 状态的分支即可。那么问题就变成了，如何让新的 CN 找到所有的事务分支呢？

其实这个信息在 prepare phase 已经知道了，那么我们可以提前（prepare phase）将这个信息写入事务日志，这样恢复的时候就有办法区分 1、2 两种情况了。为此我们调整了一下事务日志的结构：

```mysql
CREATE TABLE `GLOBAL_TX_LOG` (
  `TRX_ID` BIGINT PRIMARY KEY,
  `COMMIT_TS` BIGINT DEFAULT NULL,
  `STATE` ENUM('COMMITTED', 'ABORTED') NOT NULL,
  `PARTICIPANTS` BLOB DEFAULT NULL,
  # ...
);

# prepare phase
INSERT INTO `GLOBAL_TX_LOG` VALUES (123, NULL, "STAGED", "dn1,dn2");

# coommit phase
UPDATE `GLOBAL_TX_LOG` SET `COMMIT_TS` = 100, `STATE` = "COMMITTED";
```

基于这样一个拆分为两阶段的事务日志写入，我们就可以区分不同的情况来恢复事务了。此时我们的提交流程也发生了变化：

![async_commit_2 (1).png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/295107/1618551257577-bc96484f-74ab-4059-9afa-6112e64367ae.png)

显然这是一个非常理想的优化效果，我们在 prepare phase 以后立刻向用户返回成功，剩下的步骤都异步完成。但这会带来一个新的问题：我们的线性一致性被打破了。

假如用户在收到事务 T1 COMMIT 成功的响应以后立刻发起一新的读请求，这时候 CN 会去 TSO 获取一个属于这个新事务 T2 的 snapshot_ts，而这个有可能发生在 T1 后台获取 commit_ts 之前（即 T1 还没开始 commit phase）。这样的话 snapshot_ts_t2 < commit_ts_t1，T2 也就没法看到 T1 的写入，这是不符合基本的写后读的。导致这个现象的核心问题是，取 COMMIT_TS 发生在 COMMIT_POINT 之后的异步阶段，对用户来说这个请求的生命周期已经结束了，因此 COMMIT_TS 作为事务顺序的判断依据，也必须在 COMMIT POINT 之前确定它的值。这样我们似乎不得不把 COMMIT POINT 挪动到 UPDATE COMMIT LOG 之后（考虑到 CN 失败的情况，必须将 COMMIT_TS 持久化才认为是可靠）。仿佛一切回到了原点。

**最终方案：引入 prepare_ts**

既然 commit_ts 必须在 COMMIT POINT 之前获取并持久化，而 prepare phase 仅有的交互就是 XA PREPARE，那我们唯一的解决方案就是把 commit_ts 持久化在所有 DN 上了。这意味着我们必须在 XA  PREPARE 之前拿到 ts，这也就是 prepare_ts。

![async_commit_final.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/295107/1618552815579-60e212c8-b721-404c-b99c-584716a46baf.png)



Reference:

