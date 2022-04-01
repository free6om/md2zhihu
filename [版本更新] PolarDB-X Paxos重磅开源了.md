## 架构简介

PolarDB-X 采用 Shared-nothing 与存储分离计算架构进行设计，系统由4个核心组件组成。
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/5565/1648810112158-576e2acc-9603-4307-8442-6a624d2a1933.png)

-   计算节点（CN, Compute Node）

计算节点是系统的入口，采用无状态设计，包括 SQL 解析器、优化器、执行器等模块。负责数据分布式路由、计算及动态调度，负责分布式事务 2PC 协调、全局二级索引维护等，同时提供 SQL 限流、三权分立等企业级特性。

-   存储节点（DN, Data Node）

存储节点负责数据的持久化，基于多数派 Paxos 协议提供数据高可靠、强一致保障，同时通过 MVCC 维护分布式事务可见性。

-   元数据服务（GMS, Global Meta Service）

元数据服务负责维护全局强一致的 Table/Schema, Statistics 等系统 Meta 信息，维护账号、权限等安全信息，同时提供全局授时服务（即 TSO）。

-   日志节点（CDC, Change Data Capture）

日志节点提供完全兼容 MySQL Binlog 格式和协议的增量订阅能力，提供兼容 MySQL Replication 协议的主从复制能力。

开源地址：[https://github.com/ApsaraDB/galaxysql]

## 版本说明

我们也选择了今天给大家一份诚意满满的礼物：PolarDB-X 正式发布2.1.0版本
**本次开源包含四大核心特性，全面提升 PolarDB-X 稳定性和生态兼容性 **

### 01. 高可用的开源能力补齐

分布式一致性算法（Consensus Algorithm ）是一个分布式计算领域的基础性问题，其最基本的功能是为了在多个进程之间对某个（某些） 值达成一致（强一致），进而解决分布式系统的可用性能问（高可用），近几年NewSQL和云原生数据库的不断兴起，极大的推动了关系数据库和一致性协议的结合，常见的技术有Paxos和Raft。

**2022年4月1号，PolarDB-X 正式开源X-Paxos，基于原生MySQL存储节点，提供Paxos三副本共识协议，可以做到金融级数据库的高可用和容灾能力，做到RPO=0的生产级别可用性，可以满足同城三机房、两地三中心等容灾架构。**

Paxos协议对于面向云的架构是非常必要的，云的本质是虚拟化和资源池化，节点的变化和弹性是一个常规操作，我们需要解决面向用户透明运维的能力，任何情况下数据都不能丢、不能错。

**下面一个例子演示基于kubernetes(k8s)虚拟化，结合Paxos的高可用切换提供RPO=0**
[embed: polardb-x-paxos-demo.mp4](https://intranetproxy.alipay.com/skylark/lark/0/2022/mp4/46860/1648813273288-f74c2dff-914f-47d4-8bdf-4ec5ea550ee8.mp4)

### 02. 分布式水平扩展能力升级

PolarDB-X 作为一款基于MySQL原生分布式，除了提供基于Paxos RPO=0的金融级容灾能力外，最重要的特性就是分布式的水平扩展，在PolarDB-X 2.1.0版本正式推出新版数据分区表，提供Auto分区模式。

Auto模式的数据库支持自动分区，即创建表时无需指定分区键，数据即可自动在集群内均匀分布；同时也支持使用标准的MySQL分区表语法，对表进行手动分区。结合新版分区表能力，新增支持热点分裂、TTL(Time To Live)分区、Locality亲和性调度等能力，可以让您便捷地享受到分布式数据库的透明式分布、弹性伸缩和分区管理等诸多红利。

具体细节可参考文档：[AUTO模式数据库](https://help.aliyun.com/document_detail/416411.html)

**基于新版分区表，可扩展提供分布式热力分析能力，样例效果图 ** 
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/5565/1648813103497-eec6d315-0ed1-45f1-804b-1fe162564a19.png)

### 03. MySQL生态适配加速

PolarDB-X 架构中有一个特殊的CDC(Change Data Capture)组件，其主要用于提供分布式的增量日志获取，作为MySQL原生分布式，对应分布式CDC在设计上也选择全面兼容MySQL Binlog，在PolarDB-X 2.1.0版本我们又进一步完善了与MySQL现有CDC生态的适配和兼容。

首先，PolarDB-X CDC的binlog服务，与canal、maxwell、debezium、Flink CDC等开源MySQL binlog解析组件完成适配认证。其次，PolarDB-X CDC新增replica服务，全面兼容MySQL Replication相关协议，通过MySQL的start slave指令，可以将PolarDB-X作为开源MySQL的备库实时同步数据。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/5565/1648814207945-0ce9d840-5e9e-4f6a-8009-728f4ad840a8.png)

### 04. 轻量化部署功能完善

PolarDB-X Operator 是一个基于 Kubernetes 的 PolarDB-X 集群管控系统，希望能在原生 Kubernetes 上提供完整的生命周期管理能力，满足用户的轻量化部署。在PolarDB-X 2.1.0版本我们进一步完善了部分运维能力，比如提供Prometheus + Grafana 的监控系统、完善分布式节点升降配、扩缩容、版本升级等能力。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/5565/1648812913604-b9b2cc58-ba03-4e85-ba2a-7bae4ce37a07.png)

### 更详细的Features

-   新增 支持创建数据库指定建表模式(新的分区表模式与老的分库分表模式)，默认是分库分表模式
-   新增 支持使用 MySQL分区表语法 创建一级分区的分区表，分区策略包括Hash/Range/List等
-   新增 支持分区表的动态裁剪能力，包括支持分区列条件的常量折叠、区间合并以及前缀查询裁剪等功能
-   新增 支持分区表的JOIN计算下推
-   新增 提供分区表的分区管理能力，包括分区的添加、删除、分裂、合并与迁移等功能
-   新增 提供表组及其他能力（包括表组的创建、删除、变更等），支持分区变更期间JOIN计算下推不受影响
-   新增 支持全局索引表使用MySQL分区表语法并按Hash/Range/List等分区策略进行分区
-   新增 自动拆分支持使用分区表语法
-   新增 拆分变更增加支持分区表
-   新增 新分区表GSI自动拆分会携带主键，可以处理GSI热点问题
-   新增 支持实例的缩容
-   新增 支持分区表的TTL及其管理能力（包括调整TTL的初始时间与时间间隔等）
-   优化 Check Table 指令，支持校验主表分区、索引表分区与列定义等元数据一致性
-   新增 SQL Advisor支持推荐广播表
-   新增 支持Instant Add Column功能
-   新增 支持Explain Statistics拉取优化器优化需要的所有信息
-   新增 限制cbo的搜索空间，减少复杂查询的优化耗时
-   优化 部分DDL后台操作的数据校验任务的性能，使GSI/扩缩容DDL变更操作加速
-   新增 支持兼容MySQL的Replica相关指令
-   新增 支持存储节点PAXOS三节点集群
-   新增Replica组件，支持通过change master … 语法的方式将PolarDB-X作为MySQL Slave来消费数据
-   全局Binlog中支持记录Rows_query_event类型数据，前置条件：需将DN节点binlog_rows_query_log_events参数设置为On
-   新增 Flink CDC 接入
-   新增 CR PolarDBXMonitor 用来监控 PolarDBXCluster
-   新增 Helm Chart polardbx-monitor，包含定制化的 kube-prometheus 和预定义的 Dashboard 用来展示 PolarDB-X 集群监控信息
-   PXD 工具支持单副本和三副本两种部署模式

### ChangeLog 列表

[1] [CN ChangeLog](https://github.com/ApsaraDB/galaxysql/releases/tag/galaxysql-5.4.13)
[2] [DN ChangeLog](https://github.com/ApsaraDB/galaxyengine/commits/main)
[3] [CDC ChangeLog](https://github.com/ApsaraDB/galaxycdc/releases/tag/galaxycdc-5.4.13)
[4] [Kube ChangeLog](https://github.com/ApsaraDB/galaxykube/tree/main/CHANGELOG)



Reference:

