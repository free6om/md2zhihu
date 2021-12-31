GalaxyCDC 是云原生分布式数据库系统 PolarDB-X 的一个核心组件，负责全局增量日志的生成、分发和订阅。通过GalaxyCDC，PolarDB-X 数据库可以对外提供完全兼容 MySQL Binlog 格式和协议的增量日志，可实现与 MySQL Binlog 下游生态工具的无缝对接。本篇将对GalaxyCDC的代码结构做一个系统介绍，并讲解一下如何快速搭建开发环境。

## 总体架构

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/304384/1640672830055-9b394068-b6f8-47dd-bcdb-afe8e8b8ee01.png)

如上图所示，PolarDB-X 包含 4 个核心组件，CN (Compute Node) 负责计算、DN (Data Node) 负责存储、GMS (Global Meta Service) 负责管理元数据和提供TSO服务，CDC (Change Data Capture) 负责生成变更日志。其中，CDC作为Binlog日志数据的出口，主要负责3方面的工作：

1.  基于TSO，对各个DN的原始物理Binlog进行排序和归并，构建出全局有序的事务队列
1.  将事务队列中的数据，持久化到存储介质，生成兼容MySQL Binlog格式的逻辑Binlog文件
1.  提供兼容MySQL Dump协议的消费订阅能力，对外提供Replication服务

更加形象的来理解，GalaxyCDC是一个Streaming系统，InputSource是各个DN的原始Binlog，Computing逻辑是一系列计算规则和处理算法(排序、过略、整形、合并等等)，TargetSink是屏蔽了分布式数据库内部细节的全局逻辑Binlog文件，并提供了兼容MySQL生态的Replication能力。其运行时状态图，如下所示：

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/304384/1640674690030-217cf904-34a3-4df8-a93f-126c07add010.png)

从技术架构的角度来看，CDC共包含3个核心组件，分别是Daemon、Task和Dumper，如下所示

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/304384/1640750823011-540a62d6-41ae-4d80-8ddf-e8003d26805b.png)

1.  一个Node代表一个运行节点，可以是一台物理机、一台ECS或一个Docker容器，可以部署1到n个Node，但如果需要HA能力，则至少需要部署两个
1.  每个Node上都会运行一个Daemon进程，该进程负责进行监控、管控、调度等功能，其定位可以和Hadoop中的NodeManager类比。多个Daemon进程中会有一个Leader角色，通过抢占的方式来获取，负责一些中心化的控制功能
1.  Dumper负责从Task拉取经过处理后的binlog数据，将数据持久化到逻辑Binlog文件，并对外提供兼容Mysql Dump协议的Replication服务。每个Node上都会运行一个Dumper进程，其中有一个Dumper会被选定为Master角色，其余Dumper的角色为Slave，Slave会实时复制Master的全局逻辑Binlog文件，当发生故障时，Daemon会负责完成Master的故障转移
1.  Task是CDC的内核，负责事务排序、数据整形、事务合并、DDL处理、元数据生命周期管理等功能，整个CDC集群只会有一个Task进程，调度程序会选择负载最低的Node运行Task，并尽量保证Dumper Master和Task在不同的Node

## 工程结构

GalaxyCDC工程代码托管在GitHub上( [仓库地址](https://github.com/ApsaraDB/galaxycdc))，遵循Apache License 2.0协议。GalaxyCDC是一个多模块的 Java 项目，模块之间通过接口暴露服务，模块关系记录在 [pom.xml](https://github.com/ApsaraDB/galaxycdc/blob/main/pom.xml)中，通过 mvn dependency:tree 命令可以查看全部依赖。

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/304384/1640514416065-ac05a45d-f190-4cb0-9f8a-1f75ebca1f04.png)

如上图所示，整个项目共包含15个目录模块(除此之外，CDC在GalaxySQL项目中还有部分功能模块，具体可参见[附3])，下面对每个模块展开介绍

#### [模块]
 codestye

提供了一份内置的代码风格配置文件，主要面向IDEA，如果有兴趣贡献源码，可基于该style文件进行设置

#### [模块]
 docker

提供了一系列docker相关的配置文件，通过运行该目录下的[build.sh](https://github.com/ApsaraDB/galaxycdc/blob/main/docker/build.sh)，可以在本地快速构建出一份dokcer镜像

#### [模块]
 polardbx-cdc-assemble

程序构建模块，提供了构建/打包等相关的配置，在GalaxyCDC工程的根目录下执行mvn clean package -Dmaven.test.skip=true，会在assemble的target目录下生成polardbx-binlog.tar.gz程序包

#### [模块]
 polardbx-cdc-canal

binlog数据解析模块，引用了开源[Canal](https://github.com/alibaba/canal)的源码，并做了很多定制化改造，核心代码介绍如下：
|  包(文件)名称 |  功能简介 | 
| -------- | -------- |
| com.aliyun.polardbx.binlog.canal.binlog     |  定义了对应MySQL Binlog Event的各种数据模型，以及对Binlog二进制数据进行Decode的核心实现    |
| com.aliyun.polardbx.binlog.canal.core     | Canal的运行时内核模块，包含数据解析流程的管理、ddl的处理、权限的管理、HA的处理等等  |

#### [模块]
 polardbx-cdc-common

公共类库模块，提供了基础工具、领域模型、数据访问、公共配置等，核心代码介绍如下：

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tr class="header">
<th>包(文件)名称</th>
<th>功能简介</th>
</tr>
<tr class="odd">
<td>com.aliyun.polardbx.binlog.domain</td>
<td>包含了领域模型类的定义，其中po包下的源码为代码生成工具自动生成，对应了CDC的元数据表</td>
</tr>
<tr class="even">
<td>com.aliyun.polardbx.binlog.dao</td>
<td>包含了所有的数据访问层类定义，也是代码生成工具自动生成</td>
</tr>
<tr class="odd">
<td>com.aliyun.polardbx.binlog.heartbeat</td>
<td>包含了TsoHeartbeat组件，保证binlog stream不断向前流动，也是混合事务策略场景下生成虚拟TSO的必要条件</td>
</tr>
<tr class="even">
<td>com.aliyun.polardbx.binlog.leader</td>
<td>提供了一个基于MySQL GET_LOCK函数实现的LeaderElector</td>
</tr>
<tr class="odd">
<td>com.aliyun.polardbx.binlog.rpc</td>
<td>定义了Task和Dumper之间进行数据交互的Rpc接口</td>
</tr>
<tr class="even">
<td>com.aliyun.polardbx.binlog.scheduler</td>
<td>包含了集群调度和资源分配相关的功能实现</td>
</tr>
<tr class="odd">
<td>com.aliyun.polardbx.binlog.task</td>
<td>包含了Task和Dumper组件定时心跳相关的功能实现</td>
</tr>
<tr class="even">
<td><a href="https://github.com/ApsaraDB/galaxycdc/blob/main/polardbx-cdc-common/src/main/resources/generatorConfig.xml">generatorConfig.xml</a></td>
<td>代码生成器配置文件，使用了MyBatis Generator进行自动化代码生成</td>
</tr>
</table>

#### [模块]
 polardbx-cdc-daemon

守护进程模块，提供了HA调度、TSO心跳、监控信息收集、OpenAPI访问等功能，核心代码介绍如下：

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tr class="header">
<th>包(文件)名称</th>
<th>功能简介</th>
</tr>
<tr class="odd">
<td>com.aliyun.polardbx.binlog.daemon.cluster com.aliyun.polardbx.binlog.daemon.schedule</td>
<td>包含了集群管控相关功能 ，心跳、运行时拓扑调度、HA探活等</td>
</tr>
<tr class="even">
<td>com.aliyun.polardbx.binlog.daemon.rest</td>
<td>包含了若干Rest风格的接口，报警事件收集、Metrics收集、系统参数设置等</td>
</tr>
</table>

#### [模块]
 polardbx-cdc-dumper

全局Binlog文件dump模块，接收Task模块输入的binlog数据并落盘，并对外提供兼容MySQL dump协议的数据订阅服务，核心代码介绍如下：

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tr class="header">
<th>包(文件)名称</th>
<th>功能简介</th>
</tr>
<tr class="odd">
<td>com.aliyun.polardbx.binlog.dumper.dump.client</td>
<td>Dumper主备之间进行数据同步，对应的Client实现</td>
</tr>
<tr class="even">
<td>com.aliyun.polardbx.binlog.dumper.dump.logfile</td>
<td>逻辑Binlog文件构建模块，最为核心的两个类分别是LogFileGenerator和LogFileCopier，分别负责Dumper Master和Dumper Slave的逻辑Binlog文件处理</td>
</tr>
</table>

#### [模块]
 polardbx-cdc-format

数据整形模块，对物理binlog进行格式转换(增加列，删减列，物理库表转化为逻辑库表名称等)，保证转换后的数据格式和逻辑库表Schema始终保持一致的状态

#### [模块]
 polardbx-cdc-meta

元数据管理模块，以时间线为基准，对PolarDB-X的所有逻辑库表和物理库表进行历史版本维护，可以构建出给定任一时间点的Schema快照，是DDL处理和Binlog整形的基础支撑。此外，该模块还维护了CDC系统库表的Sql脚本定义(src/main/resources/db/migration)，CDC使用Flyway进行表结构的管理

#### [模块]
 polardbx-cdc-monitor

监控模块，一个内置的监控实现，用来管理和维护监控事件。在运行态，监控信息会通过该模块发送到daemon进程

#### [模块]
 polardbx-cdc-protocol

数据传输协议定义模块，cdc使用gRpc和protobuf来进行数据的传输，此模块定义了数据协议接口和数据交换格式，核心代码介绍如下：

<table>
<tr class="header">
<th>包(文件)名称</th>
<th>功能简介</th>
</tr>
<tr class="odd">
<td>com.aliyun.polardbx.binlog.protocol</td>
<td>包含了Task节点和Dumper节点间的数据交换格式定义</td>
</tr>
<tr class="even">
<td>com.aliyun.polardbx.rpc.cdc</td>
<td>包含了Dumper节点和CN节点间的数据交换格式定义</td>
</tr>
</table>

#### [模块]
 polardbx-cdc-storage

数据存储模块，基于rocksdb进行了封装扩展，当内存资源不足时(如大事务、big column等场景)，用来将内存数据转存到磁盘

#### [模块]
 polardbx-cdc-task

核心任务模块，可以认为是CDC的kernel，最核心的业务逻辑诸如事务的排序、整形、合并、DDL的处理、元数据的生命周期维护等操作，均在此模块完成。核心代码介绍如下：

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<tr class="header">
<th>包(文件)名称</th>
<th>功能简介</th>
</tr>
<tr class="odd">
<td>com.aliyun.polardbx.binlog.extractor</td>
<td>包含了一级排序功能，对每个DN的原始物理Binlog进行解析，并按照TSO进行排序，辅以数据整形、元数据历史版本维护、ddl处理等功能</td>
</tr>
<tr class="even">
<td>com.aliyun.polardbx.binlog.merge</td>
<td>包含了全局排序功能，接收一级排序输出的有序队列，进行多路归并，输出全局有序的事务队列</td>
</tr>
<tr class="odd">
<td>com.aliyun.polardbx.binlog.collect</td>
<td>包含了事务合并功能，从全局有序的事务队列提取数据，合并具有相同事务id的局部事务，得到完整全局事务</td>
</tr>
<tr class="even">
<td>com.aliyun.polardbx.binlog.transmit</td>
<td>包含了数据传输功能，排序合并后的数据通过传输模块发送到下游Dumper组件</td>
</tr>
</table>

#### [模块]
 polardbx-cdc-transfer

一个内置的转账程序，通过运行该转账程序，可以对CDC全局Binlog的正确性进行基本的验证，使用方法后文会有详述

## 库表说明

CDC的系统元数据表保存在GMS元数据库，下面对元数据表进行介绍

-   **binlog_system_config**

系统参数信息表，保存系统级的参数配置

-   **binlog_task_config**

系统运行时拓扑配置表，保存dumper和task的配置信息，如指定在哪个节点执行、Rpc端口值、运行内存等

-   **binlog_node_info**

节点信息表，保存了每个节点的资源配置和运行时状态等信息，每个Node对应于表中的一条记录

-   **binlog_dumper_info**

dumper运行状态信息表，每个Dumper进程对应表中一条记录

-   **binlog_task_info**

task运行状态信息表，每个Task进程对应表中一条记录

-   **binlog_logic_meta_history**

逻辑库表元数据信息历史记录表，用于记录每条逻辑DDL SQL及其对应的库表拓扑信息

-   **binlog_phy_ddl_history**

物理库表元数据信息里是记录表，用于记录每条物理DDL SQL

-   **binlog_oss_record**

binlog文件信息表，每个逻辑Binlog文件对应于表中的一条记录

-   **binlog_polarx_command**

命令信息表，CDC和CN之间部分交互通过该表来完成，如触发全量元数据的初始化等

-   **binlog_schedule_history**

调度历史记录表，集群每发生一次重新调度(如:节点宕机触发Rebalance)，会在该表中记录一条历史信息

-   **binlog_storage_history**

DN节点历史记录表，PolarDB-X进行水平扩容或者缩容时，DN数量会发生变更，每次变更都会通过“打标”的方式通知给CDC，CDC会通过该表按照时间线记录所有的变更历史。Dumper进程每次启动时会查询逻辑Binlog文件中记录的最大tso，发送给Task进程，Task会通过该tso从binlog_storage_history表中查询到当前应该连接的DN列表

-   **binlog_env_config_history**

参数变更历史表，CDC中有部分参数需要和时间线绑定，该表用来记录这些参数的变更历史，和binlog_storage_history类似，系统会以tso为基准，使用对应时间段的参数配置

-   **binlog_schema_history**

Flyway执行历史记录表

## 开发指引

下面来介绍一下，如何基于源码搭建开发环境，方便进行二次开发或代码调试

1.  JDK最低要求 1.8
1.  准备一个PolarDB-X实例(启动CN+DN即可，不要启动CDC组件)，可通过[pxd方式部署](https://polardbx.com/quickstart/topics/quickstart-pxd-cluster.html)，或通过[源码编译安装](https://polardbx.com/quickstart/topics/quickstart-development.html)
1.  准备GalaxySql源码，并执行命令mvn install -D maven.test.skip=true -D env=release ，主要是为了得到polardbx-parser包，cdc引用了该模块
1.  准备GalaxyCDC源码，并调整dev.properties中的配置，根据实际情况调整，可能需要调整的配置有

```
polardbx.instance.id
mem_size
metaDb_url
metaDb_username
metaDbPasswd
polarx_url
polarx_username
polarx_password
dnPasswordKey
```

1.  GalaxyCDC根目录下执行 mvn compile -D maven.test.skip=true -D env=dev，编译CDC源码
1.  启动Daemon进程，直接运行com.aliyun.polardbx.binlog.daemon.DaemonBootStrap即可
1.  启动Task进程，运行com.aliyun.polardbx.binlog.TaskBootStrap，并指定入参"taskName=Final"
1.  启动Dumper进程，运行com.aliyun.polardbx.binlog.dumper.DumperBootStrap，并指定入参"taskName=Dumper-1"
1.  PolarDB-X Server，执行一些Sql，观察{HOME}/binlog目录下是否会产生Binlog数据

```
create database transfer_test;
CREATE TABLE `transfer_test`.`accounts` (
    `id` int(11) NOT NULL,
    `balance` int(11) NOT NULL,
  `gmt_created` datetime not null,
    PRIMARY KEY (`id`)
) ENGINE = InnoDB DEFAULT CHARSET = utf8  dbpartition by hash(`id`) tbpartition by hash(`id`) tbpartitions 2;
INSERT INTO `transfer_test`.`accounts` (`id`,`balance`,`gmt_created`) VALUES (1,100,now());
INSERT INTO `transfer_test`.`accounts` (`id`,`balance`,`gmt_created`) VALUES (2,100,now());
INSERT INTO `transfer_test`.`accounts` (`id`,`balance`,`gmt_created`) VALUES (3,100,now());
INSERT INTO `transfer_test`.`accounts` (`id`,`balance`,`gmt_created`) VALUES (4,100,now());
INSERT INTO `transfer_test`.`accounts` (`id`,`balance`,`gmt_created`) VALUES (5,100,now());
INSERT INTO `transfer_test`.`accounts` (`id`,`balance`,`gmt_created`) VALUES (6,100,now());
INSERT INTO `transfer_test`.`accounts` (`id`,`balance`,`gmt_created`) VALUES (7,100,now());
INSERT INTO `transfer_test`.`accounts` (`id`,`balance`,`gmt_created`) VALUES (8,100,now());
INSERT INTO `transfer_test`.`accounts` (`id`,`balance`,`gmt_created`) VALUES (9,100,now());
INSERT INTO `transfer_test`.`accounts` (`id`,`balance`,`gmt_created`) VALUES (10,100,now());
```

1.  Docker 安装一个MySQL，建议安装8.0版本，提供一个安装示例如下：

```
docker run -itd --name mysql_3309 -p 3309:3306 -e MYSQL_ROOT_PASSWORD=root mysql
登录dokcer实例：docker exec -it mysql_3309  bash
编辑/etc/mysql/my.cnf，
  a. 增加如下配置来关闭Gtid (GalaxyCDC全局Binlog暂不支持Gtid)
   gtid_mode=OFF
   enforce_gtid_consistency=OFF
  b. 更改server id，避免与主库重复
   server_id = 2
重启docker实例：docker restart mysql_3309
```

1.  MySQL客户端登录新安装的MySQL，并执行如下命令，观察accounts表等信息是否已经同步到MySQL

```
stop slave;
reset slave;
CHANGE MASTER TO
    MASTER_HOST='xxx',
    MASTER_USER='xxx',
    MASTER_PASSWORD='xxx',
    MASTER_PORT=xxx,
    MASTER_LOG_FILE='binlog.000001',
    MASTER_LOG_POS=4,
    MASTER_CONNECT_RETRY=100;
start slave;
```

1.  还可以运行tranfer工程下的转账程序，进行转账场景下的测试，启动入口类：com.aliyun.polardbx.binlog.

transfer.Main，开启TSO和不开启TSO会看到不一样的现象，前者会保证强一致(即任意时刻从下游MySQL都能查询到一致的余额)，后者只保证最终一致。停止测试程序之后，可以用下面的SQL，验证两边的数据是否完全一致
SELECT BIT_XOR(CAST(CRC32(CONCAT_WS(',', id, balance, CONCAT(ISNULL(id), ISNULL(balance))))AS UNSIGNED)) AS checksum FROM accounts;

## 核心代码

下面给出CDC Stream链路中的一些核心类，这些构成了CDC运行时的骨架，提纲挈领方便快速上手

-   **负责对物理binlog进行处理的核心组件**

com.aliyun.polardbx.binlog.extractor.BinlogExtractor
com.aliyun.polardbx.binlog.extractor.filter.RtRecordFilter
com.aliyun.polardbx.binlog.extractor.filter.TransactionBufferEventFilter
com.aliyun.polardbx.binlog.extractor.filter.RebuildEventLogFilter
com.aliyun.polardbx.binlog.extractor.filter.MinTSOFilter
com.aliyun.polardbx.binlog.extractor.DefaultOutputMergeSourceHandler

-   **负责对物理Binlog进行局部排序的核心组件**

com.aliyun.polardbx.binlog.extractor.sort.Sorter

-   **负责进行全局排序&归并的核心组件**

com.aliyun.polardbx.binlog.merge.LogEventMerger

-   **负责进行事务合并的核心组件**

com.aliyun.polardbx.binlog.collect.handle.TxnShuffleStageHandler
com.aliyun.polardbx.binlog.collect.handle.TxnSinkStageHandler

-   **负责进行数据传输的核心组件**

com.aliyun.polardbx.binlog.transmit.LogEventTransmitter

-   **负责进行数据存储的核心组件**

com.aliyun.polardbx.binlog.storage.LogEventStorage

-   **负责对逻辑binlog文件进行数据写入的核心组件**

com.aliyun.polardbx.binlog.dumper.dump.logfile.LogFileGenerator

-   **负责进行Dumper主备复制的核心组件**

com.aliyun.polardbx.binlog.dumper.dump.logfile.LogFileCopier

-   **负责进行对整体运行时进行全局调度的核心组件**

com.aliyun.polardbx.binlog.daemon.schedule.TopologyWatcher

## 总结

本文主要介绍了GalaxyCDC的代码工程结构，列出了元数据表清单，展示了本地开发调试环境的搭建流程，并在最后给出了最为核心的一些功能组件。希望读者能够基于本文的介绍，并结合附录中给出的一些资料，通过实际动手，快速完成对GalaxyCDC的代码入门。后续我们会推出一系列文章，聚焦于每个点，对源码进行更为精细的解读。

## 附录

-   [附1] PolarDB-X 全局Binlog解读

https://zhuanlan.zhihu.com/p/369115822

-   [附2] PolarDB-X 全局 Binlog 解读之 DDL

https://zhuanlan.zhihu.com/p/377854011

-   [附3]PolarDB-X CN 代码结构

https://zhuanlan.zhihu.com/p/440865170

-   [附4]PolarDB-X CN 启动流程

https://zhuanlan.zhihu.com/p/443608658



Reference:

