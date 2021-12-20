## 一、什么是热点

我们平常所说的热点问题一般可以分为两类：

-   **行级更新热点**，通常是由于某些商业行为造成的，例如秒杀，聚划算之类的活动，这类热点的主要是在竞争数据的写锁，在某一时刻集中的对某几行特地的数据频繁的请求更新。
-   **分区读写热点**，在分布式数据库中数据必然要做分区，通过水平扩展的能力提升性能，但数据分区会因为一些业务模型出现一些不均衡的情况。

接上一篇 <[PolarDB-X 热点优化系列 (一) ~ 如何支持淘宝库存热点更新](https://zhuanlan.zhihu.com/p/432580272)> (重点介绍了行级更新热点的优化，主要引入事务合并提交的优化)，本文我们重点来聊聊分布式数据库下分区读写热点的相关优化。

## 二、热点是怎么产生的

在分布式数据库中，对于可能造成写入热点的情况可以归纳为以下两种：
1、造成有写入热点的第一种情况是，由于业务的需要，拆分规则选择了某些特定的列，而这个列的数据的区分度不好，造成出现数据倾斜，个别数据节点的数据量比其他节点的数据多的多。
我们以订单表为例子，谈谈热点产生的原因，该表的主键为自增的ID，表定义如下：
CREATE TABLE `orders` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `seller_id` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
)
对于这个订单表来说，如果我们为了追求数据的分布的均匀性，拆分键选择主键通过Hash的方式拆分，主键的具有唯一性，因为在PolarDB-X中，我们采取的是一致性Hash算法，所以按主键hash之后数据一定会均匀在分布在各个分区中。但是对于业务来说，通常不是按一个自增id维度去查询，业务更多的是需要频繁的按照卖家维度查询某卖家的数据，那么如下图所说，要查seller_id=88的数据，就需要做一遍全表扫描，这种查询效率是极低的。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/242865/1636861174939-3e04da7e-9525-4564-b692-9946cdb3d522.png) 
自然而然我们就想到更换一下我们的拆分键，采用seller_id通过Hash方式拆分，这种方式的好处是在按卖家维度查询数据时，我们能在优化器中利用分区裁减技术，将大部分无关分区裁减掉，仅仅扫描部分分区就可以满足业务的需求，如下图所示。但是这种拆分方式按照卖家ID拆分，相同的卖家数据会分布到相同的分区，这就会导致大卖家所在的分区数据数据异常的大，数据倾斜严重，大卖家的数据都在一个分区（例如下图中的P5），会导致这个分区出现严重的写入热点。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/242865/1636893333921-323f057d-d72d-46ae-a77d-355c2ce522d1.png) 
2、造成有写入热点的第二种情况是，目标表的拆分键具有单调属性，且是range拆分方式。例如拆分键是自增主键的，通过range拆分，那么在插入数据时，在一段时间范围内，写入总是集中在一个节点中。

## 三、PolarDB-X是如何处理写入热点的

前面我们谈过导致写入热点的原因，还是以订单表为例，按照业务的查询需求我们需要保证按照卖家id查询数据保持高效，那么必须要让数据有局部性，只能按照seller_id拆分，所以我们必须解决数据倾斜导致的热点写入的问题。

在PolarDB-X中我们的hash算法是采用一致性hash算法(具体的技术思考可参考: [PolarDB-X 数据分布解读（二） ：Hash vs Range](https://zhuanlan.zhihu.com/p/424174858))，默认是根据拆分键的hash空间（范围是[Long.Min,Long.Max)]大小和分区数，按range算法将hash空间切分成N等分的办法（也就是range(hash(partition_key))）将数据散列。

例如在上面的例子中，orders表的具体分区情况如下：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/242865/1636893510685-d9012b53-53f9-44be-a968-f3f4d2b928db.png)

PolarDB-X默认的一致性hash策略，因为一个卖家id对应一个hashcode，我们很自然的想到，可以将单个大卖家从平均的hash散列中抽取出来做独立分区，用独立的物理主机资源进行服务，这样的操作基本可以满足绝大部分的热点诉求。但作为分布式数据库的扩展性诉求来看，我们还需要进一步考虑热点大卖家单机无法支撑的情况。比如随着历史订单的逐步积累，以及业务进一步发展的诉求，我们需要支持将大卖家能打散到多台物理主机上，需要将单个大卖家分区再切分成多个子分区，必须在对seller_id拆分的基础上支持再对另一个维度的拆分，实际就是二级分区，每个二级分区可以部署到一个独立的物理主机上，从而满足热点的线性扩展的诉求。

这也是业界常用的解决数据倾斜的办法。热点分区、二级分区本质上是一个向量分区，在PolarDB-X中，我们支持与MySQL兼容的向量分区，如key分区、range columns分区、list columns分区。

作为实战演示的系列文章，我们通过一个具体的例子，为大家讲解PolarDB-X是如何在线动态来解决这类写入热点问题的。

### 第一步：通过SQL识别热点

热点优化的第一步，肯定是需要找到业务上的热点大卖家。

在PolarDB-X中，我们基于MySQL的采样技术，可以收集到各个物理分区的数据量大小，所以我们可以通过以下命令确定哪个分区的数据最大：

```
 select LOGICAL_TABLE,PHYSICAL_TABLE,PARTITION_NAME,TABLE_ROWS,PERCENT from information_schema.table_detail where schema_name='d1' and logical_table='orders';
```

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/242865/1636884372497-b09a15e7-bc75-47ee-9381-dde574ece3f1.png)

通过SQL查询我们发现p5所在的分区的数据量，占了整个表的99.66%，很明显这个分区存在严重的数据倾斜，接下来我们在确定一下这个分区有哪些卖家？各个卖家的数据量有多大？通过以下命令：

```
# 访问指定分区
select seller_id,count(1) rows_count from orders partition(p5) group by seller_id;
```

可以查到各个卖家在p5分区的数据分布情况，这个SQL仅会访问p5所在的数据分区，不会查询其他分区的数据。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/242865/1636881654791-c1a2ac8f-2a14-49de-bc19-c9fd89828bcc.png)

通过这个SQL的group by统计，我们可以发现seller_id=88的数据在p5分区占了99.8%(32768/(32768+48)), 所以seller_id=88对应的卖家是个大卖家。
目前我们是通过SQL的方式定位热点，很快我们会结合热力图，自动的发现热点，通过图形界面的方式更友好的展示出来。

### 第二步：抽取热点数据

在第一步我们知道seller_id=88的数据量在整个表中占比很高，这个卖家的查询/更新操作有可能会影响到其他卖家，为了消除热点卖家的数据对其他非热点卖家的影响，我们可以将热点卖家的数据抽取到一个单独的分区中，进而将这个独立的分区迁移到独享的DN节点，这样做的好处有两点：一是热点卖家和非热点卖家资源隔离，互不影响，二是热点卖家的数据仍在一个节点，查询或者更新避免跨分片, 性能不会比抽取前差。

```
alter tablegroup #tablegroupName extract to partition by hot value(#keyVal);
```

注意：这里alter ddl操作的主体是table group对象，这个table group会包含一组有关联关系的table，热点优化的DDL会同步变更相关的多张表，确保热点优化之后，还能保证关联表之间的join可以继续下推，另外更多关于table group对象可参见[PolarDB-X 数据分布解读（一）](https://zhuanlan.zhihu.com/p/395415647)。

如果更进一步，我们可以指定对应的大卖家分区，移动到我们指定的数据节点上，确保资源的使用

```
alter tablegroup #tablegroupName MOVE PARTITIONS #px to '#dn-4' ;
```

最后，如下图所示，我们可以看到在将seller_id=88的数据执行抽取命令后，我们对该卖家的查询还是只会访问一个数据分片。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/242865/1639670846458-1eed348e-2bb5-40d6-b87a-f436b6efbfb6.png)

### 第三步：动态变更分区键

随着seller_id=88这个卖家的数据量不断的增长，它的数据量可能突破了单DN的性能瓶颈，这时候我们需要考虑对热点数据进行散列，而不是仅仅是抽取到单独的节点，在我们的例子中，orders表采用partition by hash(seller_id)的分区方式，无法有效支持对热点卖家的二级分区，因此我们先对这个表进行拆分变更，将主键id作为第二个分区键：

```
alter table order repartition by key(seller_id,id) partitions 5;
```

这个拆分变更的ddl并没有改变order表的数据分布，没有数据迁移，分区数还是5，仅仅是将id作为第二个拆分键加进来了，仅仅修改表的分区元数据，代价是非常非常小了，执行了以上的拆分变更之后，此时我们的表的具体分区情况如下：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/242865/1636893627119-dd958c2d-9e94-44b6-b014-a4a45edef44a.png) 
将id作为第二个拆分键加进来后，尽管表的数据分布没有任何变化，但是表的hash空间却发生了质的变化，原来一个卖家id对应hash空间的一个点，现在对应一个范围, [hash(seller_id=88), Long.MIN] ~ [hash(seller_id=88), Long.MAX)，既然是个范围，那么我们就可以在这个范围内对当前分区继续切分。

### 第四步：散列热点数据

通过上一步，我们基本找到了业务上的热点，因此我们需要将这个超级卖家的热点数据做散列，期望能分布到多个分区(每个分区通过调度可以分布到多个物理节点)。因为超级卖家的热点数据，毕竟是少数的情况，我们同时期望在优化热点卖家的同时，能保持其他卖家的数据分布不变(数据仅在一个分区中)，以便提高查询的效率(减少查询的分区数)。

在PolarDB-X中，我们提供了很便利的命令，一条指令可以让我们轻松完成这个事情：

```
alter table #tableName split into partitions #N by hot value(#keyVal);
```

结合这个例子，这里#keyVal就是我们要分裂的大卖家id:88，N是将大卖家seller_id=88的数据分裂成#N个分区。执行这个分裂指令，我们看看具体的效果如何：
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/242865/1636884940242-2616d666-011f-4a42-a677-acee0236780c.png)

可以看到将seller_id=88的数据分裂成5个分区后，这5个分区的数据量基本上都是20%左右，是比较均匀的。
我们知道分裂前，如下图所示，如果我们查找seller_id=88, 肯定只会查找 P5:
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/242865/1636885139768-daef1a4b-3f39-44f2-bdd3-92eced2d268a.png)

那么热点分裂之后呢？如下图所示，如果我们再查找seller_id=88，会发现seller_id=88的查询需要访问5个分区，并不会去查找其他分区，所以我们保证数据的局部性的同时，成功的将热点数据切分为多个，解决了分区热点的读写问题。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/242865/1636885180307-cb430e42-a2f3-4986-8df1-a76644384410.png)

同时对于非热点卖家的数据，例如seller_id=99的数据，分裂前在P1，分裂后也是在P1，并没有受到影响。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/242865/1639753788419-e7801c65-f137-4afb-826a-966cf0ca182e.png)

由此可见，我们调整热点分区，并不会影响到非热点分区，大卖家的数据切分到单独的分区后我们可以进一步将其迁移到指定的数据节点，从而将大卖家的数据和普通卖家的数据在物理上完全隔离，相互不影响。

如果我们在seller_id=88的数据分裂成5个分区后，随着这个商家的数据量在整个orders表中的比例表小或者变大了，我们可以在将这个商家的数据重现切分成M（M>5或者M<5）个,例如我们在将seller_id=88的数据在且分成成5个之后，再次调整将其切分成10个,然后在将其调整为切分到一个分区，非常的灵活易用。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/242865/1636895144063-f7564d02-85ca-4dec-b1fd-5dd3d88e3b04.png)

## 四、小结

总结一下，PolarDB-X解决分区热点的方案：

1.  默认使用一致性hash分区，规避常见的顺序写入数据带来的分区热点问题，同时满足业务查询的数据局部性。
1.  允许用户在线调整hash分区的数据分布规则，将单个hash key抽取到单独的节点的方式，解决热点卖家与非热点卖家的资源隔离问题。
1.  当热点卖家数据继续膨胀，突破单节点性能后，引入动态向量分区，支持热点分区的二级散列，将单个hash key的数据散列到多个二级分区，进一步提升分布式的线性扩展。

结合PolarDB-X的online变更，不会阻塞业务，其中如何做到online？请参考我们之前的文章[PolarDB-X Online Schema Change](https://zhuanlan.zhihu.com/p/341685541) 。



Reference:

