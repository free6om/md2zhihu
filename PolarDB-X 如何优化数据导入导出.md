## 背景

数据库实际应用场景中常常需要进行数据导入导出，本文将介绍在PolarDB-X中数据导入导出的最佳实践。

## 测试环境

本文档的测试环境见下表：

<table>
<tr class="header">
<th>环境</th>
<th>参数</th>
</tr>
<tr class="odd">
<td>PolarDB-X版本</td>
<td>polarx-kernel_5.4.11-16282307_xcluster-20210805</td>
</tr>
<tr class="even">
<td>节点规格</td>
<td>16核64GB</td>
</tr>
<tr class="odd">
<td>节点个数</td>
<td>4个</td>
</tr>
<tr class="even">
<td>网络带宽</td>
<td>10 Gbps</td>
</tr>
</table>

测试的表用例：

```sql
CREATE TABLE `sbtest1` (
    `id` int(11) NOT NULL,
    `k` int(11) NOT NULL DEFAULT '0',
    `c` char(120) NOT NULL DEFAULT '',
    `pad` char(60) NOT NULL DEFAULT '',
    PRIMARY KEY (`id`),
    KEY `k_1` (`k`)
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4 dbpartition by hash(`id`);
```

## 导入导出工具介绍

​
PolarDB-X常见的数据导出方法有：

-   mysql -e命令行导出数据
-   musqldump工具导出数据
-   select into outfile语句导出数据（默认关闭）
-   Batch Tool工具导出数据  (PolarDB-X配套的导入导出工具)

PolarDB-X常见的数据导入方法有：

-   source语句导入数据
-   mysql命令导入数据
-   程序导入数据
-   load data语句导入数据
-   Batch Tool工具导入数据  (PolarDB-X配套的导入导出工具)

### MySQL原生命令使用例子

mysql -e命令可以连接本地或远程服务器，通过执行sql语句，例如select方式获取数据，原始输出数据以制表符方式分隔，可通过字符串处理改成`','`分隔，以csv文件方式存储，方法举例：

```shell
mysql -h ip  -P port -u usr -pPassword db_name -N -e "SELECT id,k,c,pad FROM sbtest1;" >/home/data_1000w.txt
## 原始数据以制表符分隔，数据格式：188092293	27267211	59775766593-64673028018-...-09474402685	01705051424-...-54211554755

mysql -h ip  -P port -u usr -pPassword db_name -N -e "SELECT id,k,c,pad FROM sbtest1;" | sed 's/\t/,/g' >/home/data_1000w.csv
## csv文件以逗号分隔，数据格式：188092293,27267211,59775766593-64673028018-...-09474402685,01705051424-...-54211554755
```

原始数据格式适合load data语句导入数据，使用方法可参考：[LOAD DATA 语句](https://help.aliyun.com/document_detail/316632.html)，使用举例：

```shell
LOAD DATA LOCAL INFILE '/home/data_1000w.txt' INTO TABLE sbtest1;
## LOCAL 代表从本地文件导入 local_infile参数必须开启
```

csv文件数据适合程序导入，具体方式可查看：[使用程序进行大数据导入](https://help.aliyun.com/document_detail/53107.html?spm=a2c4g.11186623.6.635.5c9c3227iMJCrd)。
​

### mysqldump工具使用例子

mysqldump工具可以连接到本地或远程服务器，详细使用方法可参考：[使用mysqldump导入导出数据](https://help.aliyun.com/document_detail/29687.html?spm=a2c4g.11186623.6.634.444e322748a2bX)。
导出数据举例：

```
mysqldump -h ip  -P port -u usr -pPassword --default-character-set=utf8mb4 --net_buffer_length=10240 --no-tablespaces --no-create-db --no-create-info --skip-add-locks --skip-lock-tables --skip-tz-utc --set-charset  --hex-blob db_name [table_name] > /home/dump_1000w.sql

## mysqldump导出数据可能会出现的问题：
【问题1】: mysqldump: Couldn't execute 'SHOW VARIABLES LIKE 'gtid\_mode''
【解决】：添加 --set-gtid-purged=OFF 参数关闭gtid_mode。
【问题2】:mysqldump: Couldn't execute 'SHOW VARIABLES LIKE 'ndbinfo\_version''
【解决】:查看mysqldump --version和mysql版本是否一致，使用和mysql版本一致的mysql client。
这两个问题通常是mysql client和mysql server版本不一致导致的。
```

导出的数据格式是SQL语句方式，以Batch Insert语句为主体，包含多条SQL语句，`INSERT INTO`sbtest1`VALUES (...),(...)`，net_buffer_length参数将影响batch size大小。
SQL语句格式合适的导入数据方式：

```shell
方法一：souce语句导入数据
source /home/dump_1000w.sql

方法二：mysql命令导入数据
mysql -h ip  -P port -u usr -pPassword --default-character-set=utf8mb4 db_name < /home/dump_1000w.sql
```

### Batch Tool工具使用例子

Batch Tool工具是阿里云内部开发的数据导入导出工具，支持多线程操作，使用方法可参考：[使用Batch Tool工具导入导出数据](https://help.aliyun.com/document_detail/314288.html)。
导出数据：

```shell
## 导出 默认值=分片数 个文件
java -jar batch-tool.jar -h ip  -P port -u usr -pPassword -D db_name -o export -t sbtest1 -s ,

## 导出整合成一个文件
java -jar batch-tool.jar -h ip  -P port -u usr -pPassword -D db_name -o export -t sbtest1 -s , -F 1
```

导入数据：

```shell
## 导入32个文件
java -jar batch-tool.jar -hpxc-spryb387va1ypn.polarx.singapore.rds.aliyuncs.com  -P3306 -uroot -pPassw0rd -D sysbench_db -o import -t sbtest1 -s , -f "sbtest1_0;sbtest1_1;sbtest1_2;sbtest1_3;sbtest1_4;sbtest1_5;sbtest1_6;sbtest1_7;sbtest1_8;sbtest1_9;sbtest1_10;sbtest1_11;sbtest1_12;sbtest1_13;sbtest1_14;sbtest1_15;sbtest1_16;sbtest1_17;sbtest1_18;sbtest1_19;sbtest1_20;sbtest1_21;sbtest1_22;sbtest1_23;sbtest1_24;sbtest1_25;sbtest1_26;sbtest1_27;sbtest1_28;sbtest1_29;sbtest1_30;sbtest1_31" -np -pro 64 -con 32

## 导入一个文件
java -jar batch-tool.jar -h ip  -P port -u usr -p password -D db_name -o import -t sbtest1 -s , -f "sbtest1_0" -np
```

## 导出方法对比

测试方法以PolarDB-X导出1000w行数据为例，数据量大概2GB左右。

<table style="width:100%;">
<colgroup>
<col style="width: 14%" />
<col style="width: 14%" />
<col style="width: 14%" />
<col style="width: 14%" />
<col style="width: 14%" />
<col style="width: 14%" />
<col style="width: 14%" />
</colgroup>
<tr class="header">
<th style="text-align: center;">方式</th>
<th style="text-align: center;">mysql -e命令导出原始数据</th>
<th style="text-align: center;">mysql -e命令 导出csv格式</th>
<th style="text-align: center;">mysqldump工具（net-buffer-length=10KB）</th>
<th style="text-align: center;">mysqldump工具（net-buffer-length=200KB）</th>
<th style="text-align: center;">Batch Tool工具 文件数=32（分片数）</th>
<th style="text-align: center;">Batch Tool工具 文件数=1</th>
</tr>
<tr class="odd">
<td style="text-align: center;">数据格式</td>
<td style="text-align: center;">原始数据格式</td>
<td style="text-align: center;">csv格式</td>
<td style="text-align: center;">sql语句格式</td>
<td style="text-align: center;">sql语句格式</td>
<td style="text-align: center;">csv格式</td>
<td style="text-align: center;">csv格式</td>
</tr>
<tr class="even">
<td style="text-align: center;">文件大小</td>
<td style="text-align: center;">1998MB</td>
<td style="text-align: center;">1998MB</td>
<td style="text-align: center;">2064MB</td>
<td style="text-align: center;">2059MB</td>
<td style="text-align: center;">1998MB</td>
<td style="text-align: center;">1998MB</td>
</tr>
<tr class="odd">
<td style="text-align: center;">耗时</td>
<td style="text-align: center;">33.417s</td>
<td style="text-align: center;">34.126s</td>
<td style="text-align: center;">30.223s</td>
<td style="text-align: center;">32.783s</td>
<td style="text-align: center;">4.715s</td>
<td style="text-align: center;">5.568s</td>
</tr>
<tr class="even">
<td style="text-align: center;">性能（行每秒）</td>
<td style="text-align: center;">299248</td>
<td style="text-align: center;">293031</td>
<td style="text-align: center;">330873</td>
<td style="text-align: center;">305036</td>
<td style="text-align: center;">2120890</td>
<td style="text-align: center;">1795977</td>
</tr>
<tr class="odd">
<td style="text-align: center;">性能（MB/S）</td>
<td style="text-align: center;">59.8</td>
<td style="text-align: center;">58.5</td>
<td style="text-align: center;">68.3</td>
<td style="text-align: center;">62.8</td>
<td style="text-align: center;">423.7</td>
<td style="text-align: center;">358.8</td>
</tr>
</table>

总结：

1.  mysql -e命令和mysqldump工具原理上主要是单线程操作，性能差别并不明显
1.  Batch Tool工具采用多线程方式导出，并发度可设置，能够极大提高导出性能

## 导入方法对比

测试方法以PolarDB-X导入1000w行数据为例，源数据是上一个测试中导出的数据，数据量大概2GB左右。

<table style="width:100%;">
<colgroup>
<col style="width: 10%" />
<col style="width: 10%" />
<col style="width: 10%" />
<col style="width: 10%" />
<col style="width: 10%" />
<col style="width: 10%" />
<col style="width: 10%" />
<col style="width: 10%" />
<col style="width: 10%" />
<col style="width: 10%" />
</colgroup>
<tr class="header">
<th style="text-align: center;">方式</th>
<th style="text-align: center;">source语句（net-buffer-length=10KB）</th>
<th style="text-align: center;">source语句(net-buffer-length=200KB）</th>
<th style="text-align: center;">mysql命令导入（net-buffer-length=10KB）</th>
<th style="text-align: center;">mysql命令导入（net-buffer-length=200KB）</th>
<th style="text-align: center;">load data语句导入</th>
<th style="text-align: center;">程序导入 batch-1000 thread-1</th>
<th style="text-align: center;">程序导入 batch-1000 thread-32</th>
<th style="text-align: center;">Batch Tool工具 文件数=32（分片数）</th>
<th style="text-align: center;">Batch Tool工具 文件数=1</th>
</tr>
<tr class="odd">
<td style="text-align: center;">数据格式</td>
<td style="text-align: center;">sql语句格式</td>
<td style="text-align: center;">sql语句格式</td>
<td style="text-align: center;">sql语句格式</td>
<td style="text-align: center;">sql语句格式</td>
<td style="text-align: center;">原始数据格式</td>
<td style="text-align: center;">csv格式</td>
<td style="text-align: center;">csv格式</td>
<td style="text-align: center;">csv格式</td>
<td style="text-align: center;">csv格式</td>
</tr>
<tr class="even">
<td style="text-align: center;">耗时</td>
<td style="text-align: center;">10m24s</td>
<td style="text-align: center;">5m37s</td>
<td style="text-align: center;">10m27s</td>
<td style="text-align: center;">5m38s</td>
<td style="text-align: center;">4m0s</td>
<td style="text-align: center;">5m40s</td>
<td style="text-align: center;">19s</td>
<td style="text-align: center;">19.836s</td>
<td style="text-align: center;">10.806s</td>
</tr>
<tr class="odd">
<td style="text-align: center;">性能（行每秒）</td>
<td style="text-align: center;">16025</td>
<td style="text-align: center;">29673</td>
<td style="text-align: center;">15948</td>
<td style="text-align: center;">29585</td>
<td style="text-align: center;">41666</td>
<td style="text-align: center;">29411</td>
<td style="text-align: center;">526315</td>
<td style="text-align: center;">504133</td>
<td style="text-align: center;">925411</td>
</tr>
<tr class="even">
<td style="text-align: center;">性能（MB/S）</td>
<td style="text-align: center;">3.2</td>
<td style="text-align: center;">5.9</td>
<td style="text-align: center;">3.2</td>
<td style="text-align: center;">5.9</td>
<td style="text-align: center;">8.3</td>
<td style="text-align: center;">5.9</td>
<td style="text-align: center;">105.3</td>
<td style="text-align: center;">100.8</td>
<td style="text-align: center;">185.1</td>
</tr>
</table>

总结：

1.  source语句和mysql命令导入方式，都是单线程执行SQL语句导入，实际是Batch Insert语句的运用，Batch size大小会影响导入性能。Batch size和mysqldump导出数据时的net-buffer-length参数有关。

优化点：
a、推荐将net-buffer-length参数设大，不超过256K，以增大batch size大小，来提高插入性能。
b、使用第三方类似mysqldump的工具，例如mydumper（备份）和myloader（导入）等，可多线程操作。

1.  load data语句是单线程操作，性能比mysql命令和source语句好一些。
1.  程序导入具有较好的灵活性，可自行设置合适的batch size和并发度，可以达到较好性能。推荐batch大小为1000，并发度为16~32。
1.  Batch Tool工具支持多线程导入，且贴合分布式多分片的操作方式，性能较好。

## 总结

1.  PolarDB-X兼容MySQL运维上常用的数据导入导出方法，但这些方法大多为MySQL单机模式设计，只支持单线程操作，性能上无法充分利用所有分布式资源。
1.  PolarDB-X提供Batch Tool工具，非常贴合分布式场景，在多线程操作下，能够达到极快的数据导入导出性能。



Reference:

