Batch Insert语句是常见的数据库写入数据的方式，PolarDB-X兼容MySQL协议和语法，Batch Insert语法为：

```sql
INSERT [IGNORE] [INTO] table_name(column_name, ...) VALUES (value1, ...), (value2, ...), ...;
```

影响Batch Insert性能的主要因素包括：

1.  batch size
1.  并行度
1.  分片数目
1.  列数目
1.  GSI的数目
1.  sequence数目

对于分片数目、列数目、GSI数目、sequence数目等因素是内需因素，根据实际需求进行设置，并且常常会和读性能相互影响，例如GSI数目较多情况下，写入性能肯定会下降，但是对读性能有提升。本文不详细讨论这些因素的影响，主要聚焦于batch size和并行度的合理设置。

## 测试环境

本文档的测试环境见下表：

<table>
<tr class="header">
<th>环境</th>
<th>参数</th>
</tr>
<tr class="odd">
<td>PolarDB-X版本</td>
<td>polarx-kernel_5.4.11-16279028_xcluster-20210802</td>
</tr>
<tr class="even">
<td>节点规格</td>
<td>16核64GB</td>
</tr>
<tr class="odd">
<td>节点个数</td>
<td>4</td>
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
) ENGINE = InnoDB DEFAULT CHARSET = utf8mb4;
```

## Batch特性：BATCH_INSERT_POLICY=SPLIT

​
PolarDB-X针对数据批量写入，为保障更好的并发性，对Batch Insert进行了优化，当单个Batch Insert语句大小超过256K时，PolarDB-X会将Batch Insert语句动态拆分成多个小Batch，多个小Batch之间串行执行，这个特性称为SPLIT。
​
通过BATCH_INSERT_POLICY=SPLIT的机制，在保障最佳性能的同时，减少PolarDB-X在并行执行中的Batch Insert的代价，尽可能规避分布式下多节点的负载不均衡。
​
相关参数：

1.  BATCH_INSERT_POLICY，可选SPLIT/NONE，默认值为SPLIT，代表默认启用动态拆分Batch。
1.  MAX_BATCH_INSERT_SQL_LENGTH，默认值256，单位KB。代表触发动态拆分Batch的SQL长度阈值为256K。
1.  BATCH_INSERT_CHUNK_SIZE_DEFAULT，默认值200。代表触发动态拆分Batch时，每个拆分之后的小Batch的批次大小。

关闭BATCH_INSERT_POLICY=SPLIT机制，可以添加hint参数：`/*+TDDL:CMD_EXTRA(BATCH_INSERT_POLICY=NONE)*/` 。 此参数的目标是关闭`BATCH_INSERT_POLICY`策略，这样才可以保证batch size在PolarDB-X执行时不做自动拆分，可用于验证batch size为2000、5000、10000下的性能，从测试的结果来看batch size超过1000以后提升并不明显。
​

## 单表的性能基准

​
背景：在分布式场景下单表只会在一个主机上，其性能可以作为一个基础的性能基线，用于评测分区表的水平扩展的能力，分区表会将数据均匀分布到多台主机上。
​
测试方法为对PolarDB-X中的单表进行Batch Insert操作，单表的数据只会存在一个数据存储节点中，PolarDB-X会根据表定义将数据写入到对应的数据存储节点上。

### 场景一：batch size

参数：【并行度：16】【列：4】【gsi：无】【sequence：无】

<table>
<tr class="header">
<th></th>
<th>batch size</th>
<th>1</th>
<th>10</th>
<th>100</th>
<th>500</th>
<th>1000</th>
<th>2000</th>
<th>5000</th>
<th>10000</th>
</tr>
<tr class="odd">
<td>PolarDB-X【单表】</td>
<td>性能（行每秒）</td>
<td>5397</td>
<td>45653</td>
<td>153216</td>
<td>211976</td>
<td>210644</td>
<td>215103</td>
<td>221919</td>
<td>220529</td>
</tr>
</table>

### 场景二：并行度

参数：【batch size：1000】【列：4】【gsi：无】【sequence：无】

<table>
<tr class="header">
<th></th>
<th>thread</th>
<th>1</th>
<th>2</th>
<th>4</th>
<th>8</th>
<th>16</th>
<th>32</th>
<th>64</th>
<th>128</th>
</tr>
<tr class="odd">
<td>PolarDB-X【单表】</td>
<td>性能（行每秒）</td>
<td>22625</td>
<td>41326</td>
<td>76052</td>
<td>127646</td>
<td>210644</td>
<td>223431</td>
<td>190138</td>
<td>160858</td>
</tr>
</table>

### 测试总结

结论：对于单表的测试，推荐batch size为1000，并行度为16~32时整体性能比较好。
说明：在测试batch size为2000、5000、10000时，需要添加hint参数来关闭SPLIT特性，从测试的结果来看batch size超过1000以后提升并不明显。例子： `/*+TDDL:CMD_EXTRA(BATCH_INSERT_POLICY=NONE)*/`

## 分区表的性能基准

Batch size和并行度都会影响Batch Insert的性能，下面对这两个因素分开进行测试分析。

### 场景一：batch Size

在数据分片的情况下，由于包含拆分函数，Batch Insert语句会经过拆分函数分离values，下推到物理存储上的batch size会改变，示意图如下图所示。
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/315860/1627890382681-5274e871-7181-4e8e-83f7-a471e2a236a8.png#height=364&id=TQExT&margin=%5Bobject%20Object%5D&name=image.png&originHeight=728&originWidth=1520&originalType=binary&ratio=1&size=383488&status=done&style=none&width=760)
所以数据分片下，PolarDB-X的Batch Insert语句可以更大一些，或者尽量将同一个物理表的数据放到一个Batch Insert语句中，保证拆分函数分离values后下推到单个数据分片上的batch size较合适，以提升存储节点的性能。

#### 子场景一 (BATCH_INSERT_POLICY=SPLIT)

参数：【BATCH_INSERT_POLICY：开启】【并行度：32】【分片数：32】【列：4】【gsi：无】【sequence：无】

<table>
<tr class="header">
<th></th>
<th>batch size</th>
<th>1</th>
<th>10</th>
<th>100</th>
<th>500</th>
<th>1000</th>
<th>2000</th>
<th>5000</th>
<th>10000</th>
</tr>
<tr class="odd">
<td>PolarDB-X【分片数：32】</td>
<td>性能（行每秒）</td>
<td>12804</td>
<td>80987</td>
<td>229995</td>
<td>401215</td>
<td>431579</td>
<td>410120</td>
<td>395398</td>
<td>389176</td>
</tr>
</table>

注：batch size >= 2000时，会触发BATCH_INSERT_POLICY策略。
​

#### 子场景二 (BATCH_INSERT_POLICY=NONE)

参数：【BATCH_INSERT_POLICY：关闭】其它参数相同

<table>
<tr class="header">
<th></th>
<th>batch size</th>
<th>1000</th>
<th>2000</th>
<th>5000</th>
<th>10000</th>
<th>20000</th>
<th>30000</th>
<th>50000</th>
</tr>
<tr class="odd">
<td>PolarDB-X【分片数：32】</td>
<td>性能（行每秒）</td>
<td>431579</td>
<td>463112</td>
<td>490350</td>
<td>526751</td>
<td>549990</td>
<td>595026</td>
<td>685500</td>
</tr>
</table>

总结：

1.  `BATCH_INSERT_POLICY=SPLIT`，batch size为1000，整体性能为每秒43w行，相当于单表的两倍
1.  `BATCH_INSERT_POLICY=NODE，`本测试的values是随机分布，拆分函数是Hash，所以分布到每个分片上的数据基本均匀，理论情况下分片数*1000的batch size下性能会比较好，在最大batch size为50000时，整体性能为每秒68万行

### 场景二：并行度

判断并行度合适的标准是将PolarDB-X数据节点的CPU利用率压满或将IOPS打满，以达到较好性能，因为Batch Insert语句基本无计算，所以PolarDB-X计算节点开销基本不大，主要开销在PolarDB-X数据节点。并行度过小或者过大都会影响性能，影响并行度的值的因素：节点个数、节点规格（核数和CPU）、线程池压力等，所以并行度很难得出一个确切的数字，推荐通过实践环境进行测试，找出适合该环境的最佳并行度。
​

#### 子场景一：测试4节点下，batch size为1000的不同并行度

参数：【batch size：1000】【列：4】【gsi：无】【sequence：无】

<table>
<colgroup>
<col style="width: 9%" />
<col style="width: 9%" />
<col style="width: 9%" />
<col style="width: 9%" />
<col style="width: 9%" />
<col style="width: 9%" />
<col style="width: 9%" />
<col style="width: 9%" />
<col style="width: 9%" />
<col style="width: 9%" />
<col style="width: 9%" />
</colgroup>
<tr class="header">
<th></th>
<th>thread</th>
<th>1</th>
<th>2</th>
<th>4</th>
<th>8</th>
<th>16</th>
<th>32</th>
<th>64</th>
<th>80</th>
<th>96</th>
</tr>
<tr class="odd">
<td>PolarDB-X【分片数：32】</td>
<td>性能（行每秒）</td>
<td>40967</td>
<td>80535</td>
<td>151415</td>
<td>246062</td>
<td>367720</td>
<td>431579</td>
<td>478876</td>
<td>499918</td>
<td>487173</td>
</tr>
</table>

总结：该配置下，64～80并发时性能达到峰值，大概50w行每秒。
​

#### 子场景二：不同节点个数下的并行度测试

参数：【2节点：2CN*2DN】【batch size：20000】【列：4】【gsi：无】【sequence：无】

<table>
<tr class="header">
<th></th>
<th>thread</th>
<th>4</th>
<th>8</th>
<th>12</th>
<th>16</th>
</tr>
<tr class="odd">
<td>PolarDB-X【分片数：16】</td>
<td>性能（行每秒）</td>
<td>159794</td>
<td>302754</td>
<td>296298</td>
<td>241444</td>
</tr>
</table>

参数：【3节点：3CN*3DN】【batch size：20000】【列：4】【gsi：无】【sequence：无】

<table>
<tr class="header">
<th></th>
<th>thread</th>
<th>9</th>
<th>12</th>
<th>15</th>
<th>18</th>
</tr>
<tr class="odd">
<td>PolarDB-X【分片数：24】</td>
<td>性能（行每秒）</td>
<td>427212</td>
<td>456050</td>
<td>378420</td>
<td>309052</td>
</tr>
</table>

参数：【4节点：4CN*4DN】【batch size：20000】【列：4】【gsi：无】【sequence：无】

<table>
<tr class="header">
<th></th>
<th>thread</th>
<th>16</th>
<th>32</th>
<th>40</th>
<th>64</th>
</tr>
<tr class="odd">
<td>PolarDB-X【分片数：32】</td>
<td>性能（行每秒）</td>
<td>464612</td>
<td>549990</td>
<td>551992</td>
<td>373268</td>
</tr>
</table>

总结：节点数增加，最佳性能的并行度也需要增大。2节点下峰值为8并发30w、3节点下峰值为12并发45w、4节点下峰值为32并发55w，整体随着节点数的性能提升线性率为0.9~1左右。

#### 子场景三：不同节点规格下的并行度测试

参数：【batch size：20000】【列：4】【gsi：无】【sequence：无】

<table>
<tr class="header">
<th></th>
<th>thread</th>
<th>4</th>
<th>8</th>
<th>10</th>
<th>12</th>
<th>16</th>
</tr>
<tr class="odd">
<td>PolarDB-X【4核16GB】</td>
<td>性能（行每秒）</td>
<td>165674</td>
<td>288828</td>
<td>276837</td>
<td>264873</td>
<td>204738</td>
</tr>
</table>

<table>
<tr class="header">
<th></th>
<th>thread</th>
<th>8</th>
<th>10</th>
<th>12</th>
<th>16</th>
</tr>
<tr class="odd">
<td>PolarDB-X【8核32GB】</td>
<td>性能（行每秒）</td>
<td>292780</td>
<td>343498</td>
<td>315982</td>
<td>259892</td>
</tr>
</table>

<table>
<tr class="header">
<th></th>
<th>thread</th>
<th>16</th>
<th>32</th>
<th>40</th>
<th>64</th>
</tr>
<tr class="odd">
<td>PolarDB-X【16核64GB】</td>
<td>性能（行每秒）</td>
<td>464612</td>
<td>549990</td>
<td>551992</td>
<td>373268</td>
</tr>
</table>

总结：节点规格升级，最佳性能的并行度也需要增大。4核16GB下峰值为8并发28w、8核32GB下峰值为10并发34w、16核64GB下峰值为32并发55w，整体随着节点规格的性能提升线性率为0.5~0.6左右。
​

## 测试总结

-   默认情况下，batch size建议为1000，并发度建议16~32并发，整体实例的并发性能和资源负载会比较优。
-   追求数据批量导入的最大速度，可增大batch size，建议batch size = 分片数 * [100 ~ 1000]，比如20000~50000。对应的单条Batch语句的大小控制2MB～8MB（默认最大包大小为16MB），同时需要通过hint设置BATCH_INSERT_POLICY=NONE。注意点：单条sql语句过大时，分布式下单个计算节点的压力会偏重，首先会带来一定的内存消耗风险，其次可能会出现多个节点之间的压力不均衡。
-   Batch的批量导入，消耗更多的是IOPS的资源，CPU/MEM不是主要瓶颈。因此，如果需要做资源升配来提升性能时，可以优先考虑扩容节点数，其次再考虑升配单节点规格。
-   线下文本数据的批量导入，建议使用PolarDB-X配套的导入导出工具Batch Tool，该工具即将开源，使用可参考：[使用Batch Tool工具导入导出数据](https://help.aliyun.com/document_detail/314288.html)



Reference:

