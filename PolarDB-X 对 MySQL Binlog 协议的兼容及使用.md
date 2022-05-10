# 前言

MySQL Binlog (The Binary Log) 作为MySQL的重要组件，在早期版本MySQL 3.23.14开始引入。可见其重要性，经过多个版本的迭代，已经成为MySQL不可或缺的一部分。其重要性，重要体现在以下两个方面：

1.  MySQL主从复制，使MySQL具备读写分离以及集群扩展的能力。
1.  特定场景下的数据恢复，从某个备份开始精确恢复数据，或者使用binlog的记录，用于生成逆向SQL或者数据。

# 背景

PolarDB-X 因为其CN DN的分布式架构，导致其生成的binlog也分布在各个DN上，在此背景下，PolarDB-X 从5.4.11版本开始，通过CDC组件提供了一份集中式的逻辑Binlog，由PolarDB-X的CN节点基于这份逻辑binlog对外提供dump协议支持。

![架构](https://pic1.zhimg.com/80/v2-5b069f198f6b94cfab3dbaac0ddc5ecc_1440w.jpg)

<center>**全局 Binlog 实现原理**</center>

# 原理

## Slave发起 COM_BINLOG_DUMP 指令

```
1              [12] COM_BINLOG_DUMP
4              binlog-pos
2              flags
4              server-id
string[EOF]    binlog-filename
```

Slave发起COM_BINLOG_DUMP 指令后，master会响应以下内容

-   binlog network stream
-   a ERR_Packet
-   or (if BINLOG_DUMP_NON_BLOCK is set) with EOF_Packet

通常在连接正常和参数正确的情况下，我们会收到binlog network stream。即以网络包为格式的binlog文件内容。

> uint<3> packet length (The sent binlog event can be up to 2^24 - 1 - 1 data bytes) # packet length
> uint<1> packet sequence byte<1>(0 to 255) # sequence
> uint<1> OK (0) or ERR (ff) or End of File, EOF, (fe) # status
> binlog event with packet length - 1(status) # binlog event data


由于packet length的最大值为2^24 - 1(16Mbytes)，故当binlog文件中，某个binlog event超过16M时，必然会发生拆包，这时sequence的作用就体现出来了，超过16M的 binlog event 会被一直拆包，直到完成网络发送。

## 发起Binlog Dump 的准备工作

> 通常来讲，MySQL slave 发起Binlog Dump协议，slave会发送以下SQL到master(以mysql 8.0 client 为例)，获取必要的binlog相关信息，所以PolarDB-X 也需要支持以下指令。

```
SELECT UNIX_TIMESTAMP()
SELECT @@GLOBAL.SERVER_ID
SET @master_heartbeat_period= 30000000000
SET @master_binlog_checksum= @@global.binlog_checksum
SELECT @master_binlog_checksum
SELECT @@GLOBAL.GTID_MODE
SELECT @@GLOBAL.SERVER_UUID
SET @slave_uuid= '1eb7a30b-xxxx-xxxx-xxxx-0242ac120004'
```

以上指令中，session变量的设置及查询，PolarDB-X早期版本已经支持了，需要进一步支持GLOBAL全局变量的设置及查询。在PolarDB-X中，由于每个实例都有自己的唯一节点GMS（元数据中心），我们可以直接下发这些全局变量的操作到GMS，这样就解决了全局变量的设置及查询问题，目前由于技术原因，禁止GLOBAL全局变量的设置操作。

## Binlog Dump协议数据流

![MySQL Replica](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/63037/1636526112139-d32ce903-00a2-4fbc-b009-7005e8f2ff57.png)

<center>**MySQL Replication**</center>

MySQL slave的 IO Thread 发起binlog dump指令，告知master，要从哪个binlog文件和位点开始复制数据，默认从master当前最新的文件和位点开始复制数据。
一般情况下，无锁的 IO Thread 的速度要超过 SQL Thread，同时在网络不可靠的场景下，降低了IO Thread 的复杂度。
上图是标准MySQL slave 发起binlog dump 后的行为，一些开源的binlog下游工具（canal，mysql-binlog-connector 等），则是模拟slave 的 IO Thread，发起binlog dump请求，订阅binlog数据，然后做后续的消费。
![订阅](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/63037/1636537676419-0fdea513-5d83-4c37-9d14-aaefe8dacd1e.png)

<center>**PolarDB-X Binlog 及其订阅**</center>

## MySQL提供的binlog存储格式

> 标准MySQL提供两种格式的binlog文件

### 基于 SQL 语句的记录(Statement-based logging)

> 记录每条引起数据变更的原始SQL语句，并不记录真实的数据变更。


优点:大大减小了binlog的IO压力，提升MySQL的性能，主从复制时，只要从MySQL的版本兼容主MySQL的版本，即可完成数据同步。
缺点:有些数据变动，不能使用SQL表示，比如函数，存储过程及不带排序的DELETE LIMIT等。

### 基于行的记录 (Row-based logging)

> 每一行的数据变更都单独记录，记录中有变更前和变更后的完整数据


优点：所有的变更都可以被复制，是最安全的复制方式。更少的行锁，带来更高的并发能力。
缺点：生成了大量的binlog数据，带来IO压力，会影响并发及吞吐能力。特别是在数据库里面存在大字段场景的情况下，这些行记录如果发生变更，也会整体反映在binlog里面。mysqlbinlog对Row-based logging的解析不易理解，需要追加额外参数配置，对于MyISAM而言，没有行级锁，其数据同步复制过程中，行级锁的优势也无法利用到。

### 混合模式（Mixed logging）

> MySQL 5.1 开始提供了混合模式的binlog记录，默认使用Statement-based logging，在必要的场景下，自动转为使用Row-based logging。

优点：在以上两种模式中做了一些平衡，尽量避免了各自的缺点，目前已成为MySQL 8.0默认的binlog存储格式。
缺点：增加了第三方工具对binlog的解析难度，前期版本不太稳定。

## PolarDB-X 采用的binlog文件格式

对于绝大部分用户而言，主从数据不一致是不可接受的，目前PolarDB-X 采用的是基于行的记录 (Row-based logging)，这是一种安全的binlog复制方案，可以保证主从MySQL的数据完全一致。

# 使用场景

## 1. 主从复制

> 兼容MySQL的Replication协议

-   在主PolarDB-X 上创建用于数据复制的账号，例如：

```
CREATE USER 'replication'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION CLIENT, REPLICATION SLAVE ON *.* TO 'replication'@'%';
```

-   查看最新的binlog文件及位点

```
SHOW MASTER STATUS;
SHOW BINARY LOGS;
```

-   在从MySQL上注册主实例

```
CHANGE MASTER TO
  MASTER_HOST='polardbx.example.com',
  MASTER_USER='replication',
  MASTER_PASSWORD='password',
  MASTER_PORT=3306,
  MASTER_LOG_FILE='source-bin.000001',
  MASTER_LOG_POS=4,
  MASTER_CONNECT_RETRY=10;
```

-   开启数据同步

```
START SLAVE;
```

-   查看从的同步状态

```
SHOW SLAVE STATUS;
```

## 2. 数据恢复

通过mysqlbinlog工具，我们可以分析binlog文件，得到某个位点的数据变更情况。

-   INSERT 语句

```
# at 235
#211102 15:10:49 server id 1  end_log_pos 308 CRC32 0x11a24ca6 	Query	thread_id=13611	exec_time=0	error_code=0
SET TIMESTAMP=1635837049/*!*/;
SET @@session.pseudo_thread_id=13611/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1168113696/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8mb4 *//*!*/;
SET @@session.character_set_client=255,@@session.collation_connection=255,@@session.collation_server=255/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
/*!80011 SET @@session.default_collation_for_utf8mb4=255*//*!*/;
BEGIN
/*!*/;
# at 308
#211102 15:10:49 server id 1  end_log_pos 360 CRC32 0x7b12bef1 	Table_map: `d1`.`t1` mapped to number 96
# at 360
#211102 15:10:49 server id 1  end_log_pos 402 CRC32 0xf22600aa 	Write_rows: table id 96 flags: STMT_END_F

BINLOG '
eeSAYRMBAAAANAAAAGgBAAAAAGAAAAAAAAEAAmQxAAJ0MQACAw8CMAACAQGAAgEh8b4Sew==
eeSAYR4BAAAAKgAAAJIBAAAAAGAAAAAAAAEAAgAC/wABAAAAAWGqACby
'/*!*/;
### INSERT INTO `d1`.`t1`
### SET
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2='a' /* VARSTRING(48) meta=48 nullable=1 is_null=0 */
# at 402
#211102 15:10:49 server id 1  end_log_pos 433 CRC32 0xbb81a243 	Xid = 27423
COMMIT/*!*/;
```

关键SQL可以提炼为 INSERT INTO `d1`.`t1` SET @1=1, @2='a'; @1和@2 分别对应当前表的第一个和第二个字段名。

-   UPDATE语句

```
BEGIN
/*!*/;
# at 1148
#211102 15:11:21 server id 1  end_log_pos 1200 CRC32 0xd080f6f8 	Table_map: `d1`.`t1` mapped to number 96
# at 1200
#211102 15:11:21 server id 1  end_log_pos 1250 CRC32 0x63520962 	Update_rows: table id 96 flags: STMT_END_F

BINLOG '
meSAYRMBAAAANAAAALAEAAAAAGAAAAAAAAEAAmQxAAJ0MQACAw8CMAACAQGAAgEh+PaA0A==
meSAYR8BAAAAMgAAAOIEAAAAAGAAAAAAAAEAAgAC//8AAwAAAAFjAAMAAAABQ2IJUmM=
'/*!*/;
### UPDATE `d1`.`t1`
### WHERE
###   @1=3 /* INT meta=0 nullable=0 is_null=0 */
###   @2='c' /* VARSTRING(48) meta=48 nullable=1 is_null=0 */
### SET
###   @1=3 /* INT meta=0 nullable=0 is_null=0 */
###   @2='C' /* VARSTRING(48) meta=48 nullable=1 is_null=0 */
# at 1250
#211102 15:11:21 server id 1  end_log_pos 1281 CRC32 0x0b46817c 	Xid = 27428
COMMIT/*!*/;
```

关键SQL可以提炼为 UPDATE `d1`.`t1` WHERE @1=3 AND @2='c' SET @1=3, @2='C'; 可以看到不仅仅是修改了的字段被记录下来了，未修改的字段信息也同样被记录下来了，这为我们的数据还原提供了非常完整的数据镜像。

-   DELETE 语句

```
BEGIN
/*!*/;
# at 1433
#211102 15:11:31 server id 1  end_log_pos 1485 CRC32 0xb64d63ce 	Table_map: `d1`.`t1` mapped to number 96
# at 1485
#211102 15:11:31 server id 1  end_log_pos 1527 CRC32 0xc82289cc 	Delete_rows: table id 96 flags: STMT_END_F

BINLOG '
o+SAYRMBAAAANAAAAM0FAAAAAGAAAAAAAAEAAmQxAAJ0MQACAw8CMAACAQGAAgEhzmNNtg==
o+SAYSABAAAAKgAAAPcFAAAAAGAAAAAAAAEAAgAC/wADAAAAAUPMiSLI
'/*!*/;
### DELETE FROM `d1`.`t1`
### WHERE
###   @1=3 /* INT meta=0 nullable=0 is_null=0 */
###   @2='C' /* VARSTRING(48) meta=48 nullable=1 is_null=0 */
# at 1527
#211102 15:11:31 server id 1  end_log_pos 1558 CRC32 0xc67b610a 	Xid = 27431
COMMIT/*!*/;
```

关键SQL可以提炼为 DELETE FROM `d1`.`t1` WHERE @1=3 AND @2='C'; 逆向回去就是一条 INSERT INTO `d1`.`t1` SET @1=3, @2='C';

基于以上的场景，目前有多种开源工具均可生成可视化的逆向SQL，用于做数据分析及恢复。

-   [Point-in-Time (Incremental) Recovery](https://dev.mysql.com/doc/refman/5.7/en/point-in-time-recovery.html)
-   [mysqlbinlog_flashback](https://github.com/58daojia-dba/mysqlbinlog_flashback)
-   [binlog2sql](https://github.com/danfengcao/binlog2sql)
-   [阿里云PolarDB-X 的SQL闪回](https://help.aliyun.com/document_detail/108629.html)

# 兼容多种数据同步工具

<table>
<tr class="header">
<th>数据同步工具</th>
<th>兼容版本</th>
<th>使用限制</th>
</tr>
<tr class="odd">
<td><a href="https://dev.mysql.com/doc/refman/8.0/en/change-master-to.html">MySQL Slave</a></td>
<td>&gt;=5.4.11</td>
<td>目前不支持GTID模式复制</td>
</tr>
<tr class="even">
<td><a href="https://github.com/alibaba/canal">canal</a></td>
<td>&gt;=5.4.11</td>
<td>无</td>
</tr>
<tr class="odd">
<td><a href="https://help.aliyun.com/document_detail/26592.htm">dts</a></td>
<td>&gt;=5.4.11</td>
<td>无</td>
</tr>
<tr class="even">
<td><a href="https://github.com/debezium/debezium">debezium</a></td>
<td>&gt;=5.4.12</td>
<td>不支持快照能力，使用时需要关闭快照 “snapshot.mode”: “never”</td>
</tr>
<tr class="odd">
<td><a href="https://github.com/zendesk/maxwell">maxwell</a></td>
<td>&gt;=5.4.12</td>
<td>无</td>
</tr>
<tr class="even">
<td><a href="https://streamsets.com/">streamsets</a></td>
<td>&gt;=5.4.12</td>
<td>无</td>
</tr>
<tr class="odd">
<td><a href="https://github.com/shyiko/mysql-binlog-connector-java">mysql-binlog-connector-java</a></td>
<td>&gt;=5.4.11</td>
<td>无</td>
</tr>
</table>

# 限制条件

-   PolarDB-X MySQL Binlog 协议目前仅支持基于binlog文件和位点方式的主从复制，暂不支持基于GTID的主从复制。
-   PolarDB-X @@GLOBAL 全局变量的设置，目前暂不支持。如果Binlog下游消费客户端或者其他开源binlog slave 模拟工具，需要修改PolarDB-X Master的全局变量的场景，需要暂时绕开，后续会全面支持全局变量的操作。
-   PolarDB-X binlog默认开启CRC32校验，目前暂时无法关闭。
-   暂不支持已归档的binlog文件的dump。

# 更多

> 更多关于PolarDB-X Binlog的生成及使用介绍


[PolarDB-X 全局 Binlog 解读](https://zhuanlan.zhihu.com/p/369115822)
[PolarDB-X 全局 Binlog 解读之 DDL](https://zhuanlan.zhihu.com/p/377854011)
[PolarDB-X 全局 Binlog 和备份恢复能力解读](https://zhuanlan.zhihu.com/p/385608337)



Reference:

