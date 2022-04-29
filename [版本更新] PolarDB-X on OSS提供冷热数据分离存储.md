
## 架构简介

PolarDB-X 采用 Shared-nothing 与存储分离计算架构进行设计，系统由4个核心组件组成。
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/5565/1651222315672-8f0ca286-484e-4efe-a414-8a71fc06cb90.png#clientId=u5169fef4-aa4a-4&crop=0&crop=0&crop=1&crop=1&from=paste&id=SenZz&margin=%5Bobject%20Object%5D&name=image.png&originHeight=666&originWidth=1440&originalType=url&ratio=1&rotation=0&showTitle=false&size=315838&status=done&style=none&taskId=u34d7d114-2b32-4dcb-8cfd-bee1ef48a7a&title=)

-   计算节点（CN, Compute Node）

计算节点是系统的入口，采用无状态设计，包括 SQL 解析器、优化器、执行器等模块。负责数据分布式路由、计算及动态调度，负责分布式事务 2PC 协调、全局二级索引维护等，同时提供 SQL 限流、三权分立等企业级特性。

-   存储节点（DN, Data Node）

存储节点负责数据的持久化，基于多数派 Paxos 协议提供数据高可靠、强一致保障，同时通过 MVCC 维护分布式事务可见性。

-   元数据服务（GMS, Global Meta Service）

元数据服务负责维护全局强一致的 Table/Schema, Statistics 等系统 Meta 信息，维护账号、权限等安全信息，同时提供全局授时服务（即 TSO）。

-   日志节点（CDC, Change Data Capture）

日志节点提供完全兼容 MySQL Binlog 格式和协议的增量订阅能力，提供兼容 MySQL Replication 协议的主从复制能力。
开源地址：[[https://github.com/ApsaraDB/galaxysql](https://link.zhihu.com/?target=https%3A//github.com/ApsaraDB/galaxysql)]

## 版本说明

3月底，PolarDB-X 正式发布 2.1.0 版本，包含了四大核心特性，全面提升 PolarDB-X 稳定性和生态兼容性，其中包含基于Paxos的三副本共识协议。

4月底，PolarDB-X正式发布2.1.1 版本，重点推出冷热数据新特性，可以支持业务表的数据按照数据特性分别存储在不同的存储介质上，比如将冷数据存储到Aliyun OSS对象存储上。

冷热数据分离存储业务价值：在数据库使用过程中，每天有大量的数据写入和更新。然而，通常只有时间邻近的，如一个月内，甚至一周内的数据才会被频繁更新和访问。而剩下的大量数据，都默默躺在磁盘的角落中，给存储空间带来了极大的浪费，也增加了数据库维护的成本。我们将前者中提到的频繁访问数据称为**热数据**，后者则称为**冷数据**。冷数据虽然不被频繁访问，却是十分具有价值的。它记录着用户的历史数据，例如电商的历史订单、银行系统的历史交易记录等。这些访问需求对个人用户来说是低频的，但放到整个电商用户群体，或是银行用户群体中，则是一份不小的workload。冷数据的分析处理能给用户带来很多商业上的 insight，帮助用户做出决策。
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/5565/1651222975736-4a95e1b6-8685-472f-b252-f7b6771d082b.png#clientId=u5169fef4-aa4a-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=271&id=PCk9j&margin=%5Bobject%20Object%5D&name=image.png&originHeight=542&originWidth=1310&originalType=binary&ratio=1&rotation=0&showTitle=false&size=193499&status=done&style=none&taskId=u79fdfe50-60a5-4fb8-b163-39a54c46671&title=&width=655)
更多技术细节查看：[[https://zhuanlan.zhihu.com/p/477664175](https://zhuanlan.zhihu.com/p/477664175)]

## 使用例子

### 快速配置OSS存储

-   首先源码编译安装 PolarDB-X，可参考[源码编译部署](quickstart-development.md)
-   或者通过K8S构建出 PolarDB-X集群，可参考[K8S部署](http://www.github.com/ApsaraDB/galaxykube/blob/main/docs/zh/deploy/quick-start.md)

安装完成后连接PolarDB-X集群，通过如下SQL命令进行OSS存储初始化:
例子：

```
create filestorage oss with ('file_uri' = 'oss://oss-bucket-name/', 'endpoint'='oss-endpoint', 'access_key_id'='your_ak', 'access_key_secret'='your_sk');
```

-   file_uri: OSS存储uri
-   endpoint: OSS存储 end-point
-   access_key_id: Access Key
-   access_key_secret: Secret Key

注1:（关于endpoint参数）：在阿里云OSS产品界面，通过“对象存储-Bucket列表-bucket-概览”可以获取到一组endpoint（地域节点）信息。仅当在相同Region的ECS部署环境下，使用经典网络访问或vpc网络访问endpoint。否则使用外网访问endpoint。

注2：通过执行show file storage语句来验证是否配置成功。

### 本地磁盘存储配置

对于没有OSS但希望体验用户，可以通过磁盘模式模拟OSS，比如将挂载一个NFS协议的网盘
注意：file_uri为CN节点能访问的数据目录路径，如果你无法保证多个CN能够共同访问这一数据目录路径，请限制CN节点个数为1个。

通过如下命令进行本地磁盘存储初始化:
例子：

```
create filestorage local_disk with ('file_uri' = 'file://tmp/orc/');
```

建表时存储引擎改为`engine = 'local_disk'`即可使用本地磁盘存储，例如：

```
create table sbtest1 like sysbench.sbtest1 engine = 'local_disk' archive_mode = 'loading';
```

### 开始体验

注：例子中的存储引擎都会以`engine = 'oss'`展示，如果你使用的是本地存储请修改为`engine = 'local_disk'`

#### Hello World

```
create database innodb_engine partition_mode = 'auto';
use innodb_engine;

CREATE TABLE `t1` (
    `id` int(10) UNSIGNED NOT NULL,
    `k` int(10) UNSIGNED NOT NULL DEFAULT '0',
    `c` char(120) NOT NULL DEFAULT '',
    `pad` char(60) NOT NULL DEFAULT '',
    KEY `xid` (`id`),
    KEY `k_1` (`k`)
) ENGINE = 'INNODB'
PARTITION BY KEY(`id`) PARTITIONS 4;

insert into t1 values (1, 1, '1', '1'),(2, 2, '2', '2'),(3, 3, '3', '3'),(4, 4, '4', '4');

create database oss_engine partition_mode = 'auto';
use oss_engine;

-- 创建一张表结构与t1一样的OSS表oss_t1,并将数据导入到oss_t1
create table oss_t1 like innodb_engine.t1 engine = 'oss' archive_mode = 'loading';

select * from oss_t1;
```

#### TTL分区

Innodb数据自动过期并归档到OSS存储上示例
注：例子中的存储引擎都会以`engine = 'oss'`展示，如果你使用的是本地存储请修改为`engine = 'local_disk'`

```
create database ttl_test partition_mode = 'auto';
use ttl_test;

-- 创建TTL表t_order,根据gmt_modified字段过期3个月以前数据
CREATE TABLE t_order (
    id bigint NOT NULL AUTO_INCREMENT,
    gmt_modified DATETIME NOT NULL,
    PRIMARY KEY (id, gmt_modified)
)
PARTITION BY HASH(id)
PARTITIONS 4
LOCAL PARTITION BY RANGE (gmt_modified)
STARTWITH '2021-01-01'
INTERVAL 1 MONTH
EXPIRE AFTER 3
PRE ALLOCATE 3
PIVOTDATE NOW();

-- 准备数据
insert into t_order (gmt_modified) values ('2021-01-01'),('2021-02-01'),('2021-03-01'),('2021-04-01'),('2021-05-01'),('2021-06-01'),('2021-07-01'),('2021-08-01'),('2021-09-01'),('2021-10-01'),('2021-11-01'),('2021-12-01'),('2022-01-01'),('2022-02-01'),('2022-03-01'),('2022-04-01'),('2022-05-01'),('2022-06-01'),('2022-07-01'),('2022-08-01'),('2022-09-01'),('2022-10-01'),('2022-11-01'),('2022-12-01');

-- 建立一张OSS表oss_order并与t_order绑定数据归档关系
create table oss_order like t_order engine = 'oss' archive_mode = 'ttl';

-- 正常流程为后台任务自动过期数据，为了演示方便这里模拟触发数据过期
alter table t_order expire local partition;

-- 查看数据
select * from t_order;
select * from oss_order;
```

#### Sysbench

##### 数据准备

下载Sysbench压测工具包
[sysbench.tar.gz](http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/attach/183466/cn_zh/1643167239503/sysbench.tar.gz?spm=a2c4g.11186623.0.0.6a2e6c94H1oFRt&amp;file=sysbench.tar.gz)

```
解压:
tar xzvf sysbench.tar.gz
cd sysbench/

安装编译所需依赖库:
yum -y install make automake libtool pkgconfig libaio-devel mysql-devel
./autogen.sh
./configure
make -j
make install

执行命令sysbench --version，返回sysbench 1.1.0证明压测工具包安装成功。
```

##### 准备压测配置

创建配置文件sysb.conf，将PolarDB-X连接信息填入配置文件，其中配置文件以及主要参数的解读如下：

```
mysql-host='{HOST}'
mysql-port='{PORT}'
mysql-user='{USER}'
mysql-password='{PASSWORD}'
mysql-db='sysbench'
db-driver='mysql'
percentile='95'
histogram='on'
report-interval='1'
time='60'
rand-type='uniform'
```

参数说明：

-   percentile：响应时间采样的百分位；
-   histogram：是否展示响应时间分布直方图；
-   report-interval：显示实时结果的时间间隔，单位为秒；
-   time：压测时长，单位为秒；
-   rand-type：随机数的分布模式。

##### 创建数据库sysbench

```
create database sysbench partition_mode = 'auto';
```

##### 导入压测数据

```
sysbench --config-file='sysb.conf' --create-table-options='PARTITION BY KEY(`id`) PARTITIONS 4'  --tables='1' --threads='4' --table-size='100000' oltp_point_select prepare
```

##### 将导入数据转换到OSS存储引擎

```
create database oss_sysbench partition_mode = 'auto';
use oss_sysbench;
create table sbtest1 like sysbench.sbtest1 engine = 'oss' archive_mode = 'loading';
```

##### 调整压测库为oss_sysbench

修改sysb.conf中mysql-db='oss_sysbench'

##### 执行压测

```
sysbench --config-file='sysb.conf' --db-ps-mode='disable' --skip-trx='on' --mysql-ignore-errors='all'  --tables='1' --table-size='100000' --threads=4 oltp_point_select run
```

#### TPC-H

##### 建立数据库及表结构

```
create database tpch partition_mode = 'auto';
use tpch;

CREATE TABLE `customer` (
  `c_custkey` int(11) NOT NULL,
  `c_name` varchar(25) NOT NULL,
  `c_address` varchar(40) NOT NULL,
  `c_nationkey` int(11) NOT NULL,
  `c_phone` varchar(15) NOT NULL,
  `c_acctbal` decimal(15,2) NOT NULL,
  `c_mktsegment` varchar(10) NOT NULL,
  `c_comment` varchar(117) NOT NULL,
  PRIMARY KEY (`c_custkey`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 PARTITION BY HASH(c_custkey) PARTITIONS 4;


CREATE TABLE `lineitem` (
  `l_orderkey` int(11) NOT NULL,
  `l_partkey` int(11) NOT NULL,
  `l_suppkey` int(11) NOT NULL,
  `l_linenumber` int(11) NOT NULL,
  `l_quantity` decimal(15,2) NOT NULL,
  `l_extendedprice` decimal(15,2) NOT NULL,
  `l_discount` decimal(15,2) NOT NULL,
  `l_tax` decimal(15,2) NOT NULL,
  `l_returnflag` varchar(1) NOT NULL,
  `l_linestatus` varchar(1) NOT NULL,
  `l_shipdate` date NOT NULL,
  `l_commitdate` date NOT NULL,
  `l_receiptdate` date NOT NULL,
  `l_shipinstruct` varchar(25) NOT NULL,
  `l_shipmode` varchar(10) NOT NULL,
  `l_comment` varchar(44) NOT NULL,
  PRIMARY KEY (`l_shipdate`,`l_orderkey`,`l_linenumber`),
  KEY `i_l_partkey` (`l_partkey`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 PARTITION BY HASH(l_orderkey) PARTITIONS 4;


CREATE TABLE `nation` (
  `n_nationkey` int(11) NOT NULL,
  `n_name` varchar(25) NOT NULL,
  `n_regionkey` int(11) NOT NULL,
  `n_comment` varchar(152) DEFAULT NULL,
  PRIMARY KEY (`n_nationkey`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 broadcast;


CREATE TABLE `orders` (
  `o_orderkey` int(11) NOT NULL,
  `o_custkey` int(11) NOT NULL,
  `o_orderstatus` varchar(1) NOT NULL,
  `o_totalprice` decimal(15,2) NOT NULL,
  `o_orderdate` date NOT NULL,
  `o_orderpriority` varchar(15) NOT NULL,
  `o_clerk` varchar(15) NOT NULL,
  `o_shippriority` int(11) NOT NULL,
  `o_comment` varchar(79) NOT NULL,
  PRIMARY KEY (`o_orderdate`,`o_orderkey`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 PARTITION BY HASH(O_ORDERKEY) PARTITIONS 4;


CREATE TABLE `part` (
  `p_partkey` int(11) NOT NULL,
  `p_name` varchar(55) NOT NULL,
  `p_mfgr` varchar(25) NOT NULL,
  `p_brand` varchar(10) NOT NULL,
  `p_type` varchar(25) NOT NULL,
  `p_size` int(11) NOT NULL,
  `p_container` varchar(10) NOT NULL,
  `p_retailprice` decimal(15,2) NOT NULL,
  `p_comment` varchar(23) NOT NULL,
  PRIMARY KEY (`p_partkey`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 PARTITION BY HASH(p_partkey) PARTITIONS 4;


CREATE TABLE `partsupp` (
  `ps_partkey` int(11) NOT NULL,
  `ps_suppkey` int(11) NOT NULL,
  `ps_availqty` int(11) NOT NULL,
  `ps_supplycost` decimal(15,2) NOT NULL,
  `ps_comment` varchar(199) NOT NULL,
  PRIMARY KEY (`ps_partkey`,`ps_suppkey`),
  KEY `IDX_PARTSUPP_SUPPKEY` (`PS_SUPPKEY`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 PARTITION BY HASH(ps_partkey) PARTITIONS 4;


CREATE TABLE `region` (
  `r_regionkey` int(11) NOT NULL,
  `r_name` varchar(25) NOT NULL,
  `r_comment` varchar(152) DEFAULT NULL,
  PRIMARY KEY (`r_regionkey`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 broadcast;


CREATE TABLE `supplier` (
  `s_suppkey` int(11) NOT NULL,
  `s_name` varchar(25) NOT NULL,
  `s_address` varchar(40) NOT NULL,
  `s_nationkey` int(11) NOT NULL,
  `s_phone` varchar(15) NOT NULL,
  `s_acctbal` decimal(15,2) NOT NULL,
  `s_comment` varchar(101) NOT NULL,
  PRIMARY KEY (`s_suppkey`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 PARTITION BY HASH(s_suppkey) PARTITIONS 4;
```

##### 数据准备

下载TPC-H脚本工具包[tpchData.tar.gz](http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/attach/183466/cn_zh/1645604888228/tpchData%20%281%29.tar.gz?spm=a2c4g.11186623.0.0.5e672386VqadsO&amp;file=tpchData%20%281%29.tar.gz)
解压

```
tar xzvf tpchData.tar.gz
```

修改param.conf配置文件，填入PolarDB-X实例的连接信息：

```
cd tpchData/
vim param.conf
```

```
#!/bin/bash

### remote generating directory
export remoteGenDir=./

### target path
export targetPath=../tpch/tpchRaw

### cores per worker, default value is 1
export coresPerWorker=`cat /proc/cpuinfo| grep "processor"| wc -l`

### threads per worker, default value is 1
export threadsPerWorker=`cat /proc/cpuinfo| grep "processor"| wc -l`
#export threadsPerWorker=1

export hint=""


export insertMysql="mysql -h{HOST} -P{PORT} -u{USER} -p{PASSWORD} -Ac --local-infile {DB} -e"
```

具体填入的值包括：
{HOST}：主机名
{PORT}：端口号
{USER}：用户名
{PASSWORD}：密码
{DB}: 数据库名
如果希望更高效地生成数据，可调大脚本中threadsPerWorker的值。

执行脚本，生成1 GB的数据：

```
cp workloads/tpch.workload.100.lst workloads/tpch.workload.1.lst
cd datagen
sh generateTPCH.sh 1
```

可以在tpch/tpchRaw/SF1/目录下查看到生成的数据

```
ls ../tpch/tpchRaw/SF1/
customer  lineitem  nation  orders  part  partsupp  region  supplier
```

导入数据到PolarDB-X实例

```
cd ../loadTpch
sh loadTpch.sh 1
```

##### 校验数据完整性

```
select (select count(*) from customer) as customer_cnt,
 (select count(*)  from lineitem) as lineitem_cnt,
 (select count(*)  from nation) as nation_cnt,
 (select count(*)  from orders) as order_cnt,
 (select count(*) from part) as part_cnt,
 (select count(*) from partsupp) as partsupp_cnt,
 (select count(*) from region) as region_cnt,
 (select count(*) from supplier) as supplier_cnt;

+--------------+--------------+------------+-----------+----------+--------------+------------+--------------+
| customer_cnt | lineitem_cnt | nation_cnt | order_cnt | part_cnt | partsupp_cnt | region_cnt | supplier_cnt |
+--------------+--------------+------------+-----------+----------+--------------+------------+--------------+
| 150000       | 6001215      | 25         | 1500000   | 200000   | 800000       | 5          | 10000        |
+--------------+--------------+------------+-----------+----------+--------------+------------+--------------+
```

##### 将导入数据转换到OSS存储引擎

```
create database oss_tpch partition_mode = 'auto';
use oss_tpch;

create table customer like tpch.customer engine='oss' archive_mode='loading';
create table nation like tpch.nation engine='oss' archive_mode='loading';
create table region like tpch.region engine='oss' archive_mode='loading';
create table partsupp like tpch.partsupp engine='oss' archive_mode='loading';
create table part like tpch.part engine='oss' archive_mode='loading';
create table supplier like tpch.supplier engine='oss' archive_mode='loading';
create table orders like tpch.orders engine='oss' archive_mode='loading';
create table lineitem like tpch.lineitem engine='oss' archive_mode='loading';
```

##### 采集统计信息

```
analyze table customer;
analyze table lineitem;
analyze table nation;
analyze table orders;
analyze table part;
analyze table partsupp;
analyze table region;
analyze table supplier;
```

##### 测试TPC-H 22条Query

下载测试脚本[tpch-queries.tar.gz](http://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/assets/attach/183466/cn_zh/1643102726812/tpch-queries.tar.gz?spm=a2c4g.11186623.0.0.5e672386VqadsO&amp;file=tpch-queries.tar.gz)
并解压：

```
tar xzvf tpch-queries.tar.gz
```

运行脚本，执行查询并计时

```
cd tpch-queries
'time' -f "%e" sh all_query.sh {HOST} {USER} {PASSWORD} {DB} {PORT}
```

## 结尾

PolarDB-X 冷热分离存储充分利用了OSS服务成本低、容量大的优良特性，将冷数据快速高效地从在线库中剥离出来，减轻了在线数据维护压力、以及在线库的规模成本，降低了全量数据的存储成本。同时，提供与MySQL兼容的访问方式，兼顾点查(sysbench)与分析型查询(tpc-h)的性能，并支持大数据产品的接入。

参考链接

1.  [PolarDB-X On OSS QuickStart](https://github.com/ApsaraDB/galaxysql/blob/main/docs/zh_CN/quickstart-oss.md)
1.  [PolarDB-X on OSS: 冷热数据分离存储](https://zhuanlan.zhihu.com/p/477664175)



Reference:

