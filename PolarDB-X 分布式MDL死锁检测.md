## 概述

对于只涉及数据读写的事务，有可能出现“单机事务死锁”和“分布式事务的死锁”。前者MySQL提供死锁检测能力，后者PolarDB-X也提供了死锁检测能力。但哪怕没有分布式死锁检测能力，事务也会在一定时间后超时，默认50秒，危害也没有太大。
但是当分布式读写事务和DDL结合起来之后，可能会出现分布式MDL死锁的问题。MDL的死锁危害巨大，因为它不仅会阻塞当前事务，还会阻塞后续所有事务，默认超时时间是1年。要排查起来也十分麻烦，需要到多个节点拉取MDL锁信息。
总结一下分布式MDL死锁问题：范围大、时间久、排查难。一旦出现，可能导致多个表全部流量长时间跌0的危险情况。

## 问题背景

关系型数据库的事务一般都要提供ACID的保证，这就意味着DDL的执行不能干扰到其他事务的ACID特性。当DDL和其他读写事务并发执行时，一方面DDL会修改表结构，另一方面我们又希望读写事务能永远看到一致性的表结构。所以，在MySQL的早期版本中（小于5.6），直接禁止DDL和DML并发执行。这种情况一直持续到MySQL引入了MDL锁和Online DDL能力后，才有所改善。
简单来说，MySQL引入Online DDL能力后：

1.  读写事务会获取元数据的读锁（后称：MDL的S锁），DDL会获取元数据的写锁（后称：MDL的X锁）。
1.  MySQL的Online DDL会将一条DDL语句分成很多个阶段，只有在必要的阶段（通常时间会压缩地很短）才会获取MDL的X锁。所以，在DDL的大部分阶段，读写事务都是可以并发执行的，只有在那些必要的阶段，DDL才会阻塞读写事务。
1.  为了保证读写事务不会对DDL形成活锁，MDL一般都会被设计成一个“公平锁”。

即便MySQL有了Online DDL，大家还是只敢在半夜进行DDL操作。其中一个重要原因就在于MDL锁的“公平性”。当DDL在等待一个长事务时，它将阻塞后续所有的读写事务，极有可能造成业务的中断，并且MySQL获取MDL锁的超时时间默认长达一年，是一件非常危险的事情。
从锁的视角来看：MDL请求队列中的X锁，阻塞了后续S锁的申请，大家都在排队等待最前面的MDL锁释放。换句话说，MDL请求队列中的X锁，会将它之前的所有S锁升级成X锁。
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/100595/1632294654642-908eeb0d-669b-448e-a1cb-d87821a7e3d1.png#clientId=ub6881f42-39d2-4&from=paste&height=311&id=ub8084c8a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=622&originWidth=1158&originalType=binary&ratio=1&size=99652&status=done&style=none&taskId=ue23237c0-155d-4628-9b73-1027e8a4832&width=579)

## 分布式MDL死锁的形成

以下是形成“分布式MDL死锁”的SQL执行流程。

<table>
<tr class="header">
<th>分布式 Transaction1</th>
<th>分布式 Transaction2</th>
<th>DDL1</th>
<th>DDL2</th>
</tr>
<tr class="odd">
<td>xa start</td>
<td>xa start</td>
<td></td>
<td></td>
</tr>
<tr class="even">
<td>insert into t1 (c2) values (2);</td>
<td></td>
<td></td>
<td></td>
</tr>
</table>

-- 获得t1的MDL S锁 | insert into t2 (c2) values (2);
-- 获得t2的MDL S锁 |  |  |
|  |  | alter table t1 add column c5 bigint;
-- 尝试获得t1的MDL X锁，阻塞等待 | alter table t2 add column c5 bigint;
-- 尝试获得t2的MDL X锁，阻塞等待 |
| insert into t2 (c2) values (2);
-- 尝试获得t2的MDL S锁。但因为MDL是公平锁，所以被DDL2阻塞 | insert into t1 (c2) values (2);
-- MDL DeadLock
-- 尝试获得t1的MDL S锁。但因为MDL是公平锁，所以被DDL1阻塞 |  |  |

根据上述的流程，我们在MySQL中复现了分布式MDL死锁的情况：
由下图可见，XA1事务、XA2事务、DDL1语句、DDL2语句全部进入阻塞等待状态，形成了死锁。
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/100595/1632236498877-ea61aa58-5b74-4ce2-8c56-335dfb957d49.png#clientId=u25fd0bd0-5862-4&from=paste&height=400&id=ua3d44eb4&margin=%5Bobject%20Object%5D&name=image.png&originHeight=800&originWidth=1492&originalType=binary&ratio=1&size=633824&status=done&style=none&taskId=ude2277d6-aed4-49cb-936a-66130170f7b&width=746)

## 解决方案

我们可以根据事务的Wait-For关系，构造有向图，然后检测环路的方式来检测是否发生了死锁。一旦发生，则选择其中的一个线程kill掉即可。具体实施过程如下：

1.  从所有MySQL节点收集事务信息，将同一个分布式事务中多个ResourceManager(RM)的事务信息合并在一起。形成有向图中的一个节点，比如下图中的XA1、XA2。
1.  同时，也构建出所有事务之间的wait-for关系。
1.  从所有MySQL节点收集MDL信息，比如下图中的DDL1、DDL2。
1.  同时，也构建出所有DDL和事务间的wait-for关系。
1.  检测环路。比如下图中XA1->DDL2->XA2->DDL1->XA1形成了环路。
1.  根据kill策略，kill掉事务或DDL，解开死锁。

![image.png](https://cdn.nlark.com/yuque/0/2021/png/577720/1612369023134-528a4456-0354-40b5-a793-2a94d5548f14.png#height=497&id=bnO4p&margin=%5Bobject%20Object%5D&name=image.png&originHeight=994&originWidth=1336&originalType=binary&ratio=1&size=177222&status=done&style=none&width=668)

## MDL锁超时抢占

当然，分布式MDL死锁虽然危险，但发生的概率也相对较低。在生产环境更常发生的情况是MDL锁阻塞读写请求。它虽然能自愈，但还是可能导致长时间的读写请求流量跌0。
如前文所论述：

1.  如果有一个长事务一直不提交，它就会一直持有MDL的S锁
1.  此时执行一个DDL请求，它会尝试请求MDL的X锁
1.  后续所有的读写请求都会被DDL阻塞，导致流量跌0，CPU飙升

![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/100595/1632389507300-5edc5c7d-4b96-4329-953d-1d5435588358.png#clientId=ue8603406-ec4c-4&from=paste&height=297&id=ua193813a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=594&originWidth=1276&originalType=binary&ratio=1&size=444968&status=done&style=none&taskId=uf3bddc84-2235-4aab-ba6c-4cee00541cb&width=638)
针对上述场景，PolarDB-X会允许DDL抢占长事务的MDL锁，避免阻塞后续的读写请求。从下图可以看到，DDL在等待了一段时间后执行成功了，而长事务被KILL掉。
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/100595/1632390829441-62ff6fe3-63ff-4ad2-bad2-8f45c3c71ef0.png#clientId=ue8603406-ec4c-4&from=paste&height=195&id=ud4d71e21&margin=%5Bobject%20Object%5D&name=image.png&originHeight=390&originWidth=2280&originalType=binary&ratio=1&size=269719&status=done&style=none&taskId=u0ffd4391-77a0-47e7-b0f2-abcc56a86bf&width=1140)

## 总结

分布式MDL死锁相比于普通的数据死锁，危害巨大并且难以排查。一旦出现，哪怕经验丰富的DBA和开发者都难以短时间内解决问题。
作为一款致力于“让用户做DDL的时候能更任性”的数据库，PolarDB-X为DDL的online能力、Crash Safe能力、性能等都做了很多的优化，欢迎持续关注我们的文章。
​

## 参考资料

1.  PolarDB-X 让“Online DDL”更Online



Reference:

