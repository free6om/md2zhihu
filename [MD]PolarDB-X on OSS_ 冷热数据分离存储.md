在即将发布的PolarDB-X 5.4.14版本中，我们将基于OSS存储服务，推出冷热数据分离存储这一新功能。在这一功能的基础上，您可以便捷地将冷数据从源表中剥离出来，归档至更低成本的OSS中，形成一张归档表；归档表支持高效的主键与索引点查、复杂分析型查询，满足高可用、MySQL兼容性和任意时间点闪回等特性。您可以像访问MySQL表一样来访问归档表，也可以用开源大数据产品接入OSS的归档数据。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/46860/1646726131885-e1212b2a-637f-415c-9aa3-16f9ddd110bd.png)

# 为什么需要冷热分离？

在数据库使用过程中，每天有大量的数据写入和更新。然而，通常只有时间邻近的，如一个月内，甚至一周内的数据才会被频繁更新和访问。而剩下的大量数据，都默默躺在磁盘的角落中，给存储空间带来了极大的浪费，也增加了数据库维护的成本。我们将前者中提到的频繁访问数据称为**热数据**，后者则称为**冷数据**。

通过对多位大型政企客户的走访和交流，我们感受到了开发者们对于冷热分离存储的迫切需求。何谓冷热分离？从字面意义上来理解，就是将热数据保留在高性能的存储设备中，用于应对日常频繁的写入与更新，满足用户对事务型数据处理的需要；冷数据则被迁移到低成本的存储设备里（这一过程也被称为“**归档**”），减轻热数据的维护压力，提供查询和局部订正的功能。

虽然不被频繁访问，冷数据却是十分具有价值的。它记录着用户的历史数据，例如电商的历史订单、银行系统的历史交易记录等。这些访问需求对个人用户来说是低频的，但放到整个电商用户群体，或是银行用户群体中，则是一份不小的workload。冷数据的分析处理能给用户带来很多商业上的 insight，帮助用户做出决策。因此还需要支持在线分析型数据处理的能力。跨越冷热数据的Join（连接）、Aggregation（聚合）是开发者们经常使用的分析手段。
因此，在PolarDB-X的冷热分离存储设计中，我们兼顾了高性能的点查和分析型查询，来满足不同用户对冷数据的访问需求。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/46860/1646726191163-3388934c-f0e0-41e3-9294-82ace108afc8.png)

# 为何选择OSS?

阿里云对外提供两类云存储服务：块存储与对象存储。其中块存储如ESSD等，是数据库事实上采取的存储方式，配备了RDMA网络服务和高性能SSD盘来提升访问性能；而对象存储如OSS，则利用低廉的HDD盘和标准网络，对外提供低成本、海量空间的存储服务。

PolarDB-X数据库原本的存储方式采用了Paxos三副本高可用集群，格式为InnoDB行存。在冷热分离存储架构中，我们将冷数据迁移到阿里云OSS对象存储中，并采用开源列存格式ORC。阿里云的OSS 对象存储服务本身保障了 12个9 的高可用性，因此我们采用了单副本的存储方式，这与 paxos 的三副本有所不同。

结合OSS单位存储的低成本，和ORC格式本身的压缩比，我们可以得到下列一组对比数据，来形成直观的感受：

| 
 | **ORC 列存 on OSS** | **InnoDB 行存** |
| --- | --- | --- |
| **存储单价** | 0.12 元/GB/月（本地冗余）
0.15 元/GB/月（同城冗余） | 0.72 元/GB/月（物理机 SSD） |
| **副本数** | 1（底层多副本对用户透明） | 3（Paxos 三副本） |
| **压缩比** | 0.20x
（实测 lineitem SF=100 占用空间 15.6GB） | 1.55x
（实测lineitem SF=50 占用空间 61GB） |
| **最终单价** | 0.024 元/GB/月 | 1.12 元/GB/月 |

# 优势特性

## TTL（time-to-live）

如何将冷数据从InnoDB行存中剥离出来？这是一个令很多开发者头疼的问题。如果使用delete from 语句 + where条件的形式来删除冷数据，很可能会因为扫描行数太多、数据太过分散，而造成锁表，影响整个数据库实例的访问；如果提前按照时间进行分区，再逐个将旧时间分区drop掉，则许多不适合按照时间分区的表将会束手无策。
针对用户反馈的这一实际问题，PolarDB-X 引入了TTL(time-to-live)这一新特性来帮助用户完成冷热数据剥离。用户无需手动维护，而是通过提前指定起始时间、分区大小和过期时间等信息，来完成数据的自动过期。我们在更底部的存储层将每张物理表做进一步的透明分区，数据按照最近的更新时间被集中到一起。
例如对于订单表t_orders，用户按照订单id进行哈希分区。引入了TTL之后，每个分区被进一步透明划分。旧时间分区（图中的2022-01分区）的过期，如同撕掉便利贴一样，在不锁表、不手动分区的情况下完成冷热数据的剥离。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/46860/1646726328587-b939d42e-6407-4a2d-89b5-885abf9c0a54.png)

关于TTL的具体使用，可以参考官网文档：什么是[TTL功能](https://help.aliyun.com/document_detail/403528.html) ？

## 高性能查询

当冷数据从主库中剥离出来，归档至OSS存储服务后，我们就得到了一张以OSS为存储载体的归档表。它完全兼容MySQL数据类型和各种查询方式，在低成本、高可用的前提下，能带来与主表一致的使用体验。
为了满足不同用户对历史数据的查询需要，我们在设计上兼顾了点查和复杂分析型查询。对此我们进行了相应的测评。由于PolarDB-X on OSS 使用列存，在报表查询中有天然的优势，因此相比于PolarDB-X on MySQL 行存模式，TPC-H测试成绩有了大幅提升；1亿行数据量下的Sysbench点查测试也显示，归档表可以满足历史数据的查询要求。
在实现以上功能的过程中，最为关键的设计是文件系统、多级缓存、多级索引与查询裁剪。此外还包括列存索引选择、向量化计算、AGG加速等，我们都将在后续的文章中详细介绍。

### TPC-H性能测试

**规格**：

-   CPU：6 * 16C
-   内存：6 * 128GB
-   SF = 100 （TPC-H 100GB）

总耗时约89s （PolarDB-X on MySQL 总耗时 150s）

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/46860/1646726434642-e073394d-31b6-4d6f-9e66-151ad2b249e0.png)

### Sysbench 性能测试

**规格**：

-   压测ECS：1 * 8C32G
-   CN：6 * 16C128G
-   Sysbench表行数： 1亿
-   并发数：100

sysbench性能测试数据如下：

<table>
<tr class="header">
<th><strong>查询类型</strong></th>
<th><strong>QPS</strong></th>
<th><strong>RT(95th percent) 单位: ms</strong></th>
</tr>
<tr class="odd">
<td><strong>主键点查</strong></td>
<td>21762.46</td>
<td>6.79</td>
</tr>
<tr class="even">
<td><strong>主键范围查询</strong></td>
<td>10769.62</td>
<td>167.44</td>
</tr>
<tr class="odd">
<td><strong>二级索引点查</strong></td>
<td>2760.82</td>
<td>55.82</td>
</tr>
</table>

## 一键迁移

完成了冷热数据剥离后，如何将数据快速归档到OSS上呢？我们基于MySQL标准语法，提供了非常简易便捷的方式，只需要执行一条建表语句：

```sql
 CREATE TABLE [oss_table_name] LIKE [innodb_table_name] 
 ENGINE = 'OSS' ARCHIVE_MODE = 'TTL'
```

执行后，OSS表将克隆InnoDB表的表结构，免去用户对归档表结构的设计；同时，冷数据归档表和源表被绑定起来，源表过期的数据将自动导入到归档表中。此后，用户可以像访问普通表一样，通过SQL来完成包括点查、范围查询、复杂分析型查询在内的各种数据访问。

### 手动强制过期

如果您想要更灵活的过期和归档操作，下列语句可以让您手动过期数据，并将过期数据导入至OSS中：

```sql
ALTER TABLE [innodb_table_name] EXPIRE LOCAL PARTITION [local_partition_name]
```

## 更多特性

### 任意时间点备份与闪回

在阿里云官方售卖的PolarDB-X 企业版中，支持了冷数据多副本备份与异地容灾。此外，PolarDB-X 将OSS归档表的版本控制与TSO结合起来，支持将整张表恢复到任意时间点之前的状态，也支持通过指定时间点来完成快照读。
您可以使用下列的闪回语句，让整张OSS归档表回到任意时间点之前的状态：

```sql
ALTER TABLE [oss_table_name] AS OF timestamp 'yyyy-mm-dd hh:mm:ss'
```

通过下列语句，指定时间点，完成在OSS归档表上的快照读：

```sql
SELECT xxx FROM [oss_table_name] AS OF timestamp '2022-01-01 01:02:03'
```

### MySQL兼容性

PolarDB-X使用开源格式ORC来作为数据存储格式。ORC起源于Hive生态，其数据类型相比于MySQL有许多受限制的地方，例如不支持高精度的Decimal、不支持Collation、时间表示范围不够大、不支持Time类型等问题。因此，在ORC格式的基础上，想要提供MySQL风格的查询体验，还需要填补这一鸿沟。
为了给用户提供与MySQL一致的使用体验，我们精心设计了一套兼容MySQL的数据类型处理方案。包括time类型支持、基于collation的字符串查找、基于字节序的Decimal数值搜索等，构建起了从Hive生态到MySQL生态的桥梁。

### 开放性

我们将提供轻量级的ORC SDK。您可以通过ORC Connector 和catalog，将OSS上存储的ORC文件作为数据源，轻松地完成Spark、Flink、Presto等开源大数据产品的接入。
在开源版本中，您还可以使用其他存储设备或服务来存放归档表，只需在执行create table时，指定Engine参数的值，如Engine = 'S3' / Engine = 'local_disk' 等，将归档表存放在S3存储服务或本地磁盘上。

# 演示视频

[demo.mp4](https://yuque.antfin.com/attachments/lark/0/2022/mp4/219382/1646028301992-4dce064f-3cb5-4136-9d53-d769f1b27061.mp4?_lake_card=%7B%22src%22%3A%22https%3A%2F%2Fyuque.antfin.com%2Fattachments%2Flark%2F0%2F2022%2Fmp4%2F219382%2F1646028301992-4dce064f-3cb5-4136-9d53-d769f1b27061.mp4%22%2C%22name%22%3A%22demo.mp4%22%2C%22size%22%3A81729032%2C%22type%22%3A%22video%2Fmp4%22%2C%22ext%22%3A%22mp4%22%2C%22status%22%3A%22done%22%2C%22taskId%22%3A%22u5d3b49b3-6e33-41d0-93d7-5e63167b54e%22%2C%22taskType%22%3A%22upload%22%2C%22id%22%3A%22udedad461%22%2C%22card%22%3A%22file%22%7D)
[![demo.mp4 (77.94MB)](https://gw.alipayobjects.com/mdn/prod_resou/afts/img/A*NNs6TKOR3isAAAAAAAAAAABkARQnAQ)]()

# 总结

PolarDB-X 冷热分离存储充分利用了OSS服务成本低、容量大的优良特性，将冷数据快速高效地从在线库中剥离出来，减轻了数据维护压力，降低了数据存储成本。同时，提供与MySQL兼容的访问方式，兼顾点查与分析型查询的性能，并支持大数据产品的接入。未来我们将在冷热数据分离这一赛道上不断前进。



Reference:

