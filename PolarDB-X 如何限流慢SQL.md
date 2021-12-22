如果用户发现活跃连接数、cpu 使用率等指标处于高位, 同时慢SQL日志中发现大量记录, 分析得出是大量慢 SQL占用了数据库资源，而且这些慢SQL已经影响到整体核心业务的稳定运行，此时我们需要对其进行限流。
本文举例说明，如何通过[限流语法](https://help.aliyun.com/document_detail/316676.html)对慢SQL进行有效限流。

## 补充信息

介绍一些和SQL限流使用相关联的一些技巧。

### 如何从会话中查看慢SQL？

可在[实例会话页面](https://help.aliyun.com/document_detail/314648.html)查看，也可使用如下指令：

```sql
select *
  from information_schema.processlist
 where COMMAND!= 'SLEEP'
   and TIME>= 1000
 order by TIME DESC;
```

### 如何观察限流规则的效果？

-   监控指标恢复情况
-   业务侧反馈
-   show ccl_rules 查看每个限流规则的限流情况的统计信息
-   查看会话和SQL日志

### 完整的涉及SQL限流的运维操作是怎样的？

1.  通过SQL日志或者会话发现慢SQL，[分析慢SQL](https://help.aliyun.com/document_detail/311122.html)
1.  创建限流规则，可使用SQL命令，或者[实例会话页面](https://help.aliyun.com/document_detail/314648.html)里“SQL限流”功能上的白屏化操作，如果用户不确定SQL限流的并发度应该是多少，可以先设置为单个计算节点的CPU核数，基于此看效果继续调整
1.  观察限流规则效果
1.  创建索引、修改SQL、增加资源等
1.  关闭限流规则，DROP CCL_RULE或者CLEAR CCL_RULES

## 案例介绍

接下来举例说明如何对发现的慢SQL进行限流，用户可参照案例中的限流规则，修改后使用。
​

### 案例1: 慢SQL属于同一个SQL模版

某DBA收到了数据库资源某指标处于高位的报警，该用户查看数据库慢日志和会话均发现有如下的慢SQL：

```sql
+--------+---------------+---------------------+--------------------+---------+------+-------+----------------------------------------------+-----------------+
| ID     | USER          | HOST                | DB                 | COMMAND | TIME | STATE | INFO                                         | SQL_TEMPLATE_ID |
+--------+---------------+---------------------+--------------------+---------+------+-------+----------------------------------------------+-----------------+
| 951494 | userxxxxxxxxx | 222.0.0.1:33830     | analy_db           | Query   |   40 |       | select * from bmsql_oorder where `o_id` > 12 | 65c92c88        |
| 952468 | userxxxxxxxxx | 222.0.0.1:33517     | analy_db           | Query   |   43 |       | select * from bmsql_oorder where `o_id` > 10 | 65c92c88        |
| 953468 | userxxxxxxxxx | 222.0.0.1:33527     | analy_db           | Query   |   43 |       | select * from bmsql_oorder where `o_id` > 23 | 65c92c88        |
| 954468 | userxxxxxxxxx | 222.0.0.1:33537     | analy_db           | Query   |   43 |       | select * from bmsql_oorder where `o_id` > 25 | 65c92c88        |
| 955468 | userxxxxxxxxx | 222.0.0.1:33547     | analy_db           | Query   |   43 |       | select * from bmsql_oorder where `o_id` > 27 | 65c92c88        |
+--------+---------------+---------------------+--------------------+---------+------+-------+----------------------------------------------+-----------------+
```

可见，这些慢SQL属于同一个SQL模版（模版ID为 **65c92c88**）：

```sql
select * from bmsql_oorder where `o_id` > ?
```

分析发现bmsql_oorder为一个数据量较大的表，而且列`o_id`上没有索引，显然这个一个未经优化的SQL，占尽了数据库资源影响会其他重要SQL的正常执行。这是一个非常适合利用模版ID去做SQL限流的场景。

**创建限流规则**

-   如果这个SQL模版的SQL不应该在当时执行，而且应该在业务低峰期执行，则我们可以创建SQL限流规则不让它执行：

```sql
CREATE CCL_RULE `KILL_CCL`       //限流规则名称为KILL_CCL
    ON `analy_db`.`*`              //&匹配analy_db下的所有表上执行的SQL
  TO 'userxxxxxxxxx'@'%'         //&匹配来自userxxxxxxxxx用户的SQL
    FOR SELECT                     //&匹配是SELECT类型的SQL语句
  FILTER BY TEMPLATE '65c92c88'  //&匹配模版ID
  WITH MAX_CONCURRENCY = 0;      //设置单节点并发度为0，不允许匹配到的SQL执行
```

客户端执行再次执行这类SQL的时候将会返回报错信息：

```
ERROR 3009 (HY000): [13172dbaf2801000][10.93.159.222:3029][analy_db]Exceeding the max concurrency 0 per node of ccl rule KILL_CCL
```

-   如果我们允许这个SQL模版的SQL少量执行，只要不占尽数据库资源就行，则我们可以在创建限流规则的时候设置一定的并发度：

```sql
CREATE CCL_RULE `KILL_CCL_2`       //限流规则名称为KILL_CCL
    ON `analy_db`.`*`              //&匹配analy_db下的所有表上执行的SQL
  TO 'userxxxxxxxxx'@'%'         //&匹配来自userxxxxxxxxx用户的SQL
    FOR SELECT                     //&匹配是SELECT类型的SQL语句
  FILTER BY TEMPLATE '65c92c88'  //&匹配模版ID 65c92c88
  WITH MAX_CONCURRENCY = 2;      //允许单个节点可以同时有两个这样的SQL在执行
```

也可使用[实例会话页面](https://help.aliyun.com/document_detail/314648.html)里“SQL限流”功能，如下操作：
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/220052/1631947463392-de3e8938-5610-4aab-8b42-d3f918452405.png#clientId=uf4700260-96d6-4&from=paste&height=606&id=u46d4f13c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=606&originWidth=871&originalType=binary&ratio=1&size=96740&status=done&style=none&taskId=u03342dbf-82d6-4a90-a16b-0bd498cae6d&width=871)
​

-   如果我们希望这个SQL模版的SQL执行的时候可以慢，但尽量不要出错，则可以设置等待队列和等待超时时间（默认为600秒）：

```sql
CREATE CCL_RULE `QUEUE_CCL_2`       //限流规则名称为KILL_CCL
    ON `analy_db`.`*`              //&匹配analy_db下的所有表上执行的SQL
  TO 'userxxxxxxxxx'@'%'         //&匹配来自userxxxxxxxxx用户的SQL
    FOR SELECT                     //&匹配是SELECT类型的SQL语句
  FILTER BY TEMPLATE '65c92c88'  //&匹配模版ID
  WITH MAX_CONCURRENCY = 2, WAIT_QUQUE_SIZE=20, WAIT_TIMEOUT=500; //单节点并发度为2，单节点等待队列长度为20，等待超时时间为500秒
```

创建完后，我们可以通过show ccl_rules指令来观察各个限流规则的实际效果，比如当前匹配到某个限流规则的正在执行的SQL的数量、被限流报错的SQL数量、总匹配成功次数等。如果想放开被限流SQL，比如在增加了某个索引后，被限流SQL的执行效率变高了，则可以通过drop ccl_rule命令来关闭指定限流规则，或者简单使用clear ccl_rules来关闭所有的限流规则。
​

当然上面的SQL也可以通过关键字来限流，将SQL语句上的关键字做拆分，我们得到关键字列表：

<table>
<tr class="header">
<th>select</th>
<th>from</th>
<th>bmsql_oorder</th>
<th>where</th>
<th><code>o_id</code></th>
</tr>
<tr class="odd">
<td></td>
<td></td>
<td></td>
<td></td>
<td></td>
</tr>
</table>

创建限流规则：

```sql
CREATE CCL_RULE `KILL_CCL`       //限流规则名称为KILL_CCL
    ON `analy_db`.`*`              //&匹配analy_db下的所有表上执行的SQL
  TO 'userxxxxxxxxx'@'%'         //&匹配来自userxxxxxxxxx用户的SQL
    FOR SELECT                     //&匹配是SELECT类型的SQL语句
  FILTER BY KEYWORD('select','from','bmsql_oorder','where','`o_id`')  //&匹配模版ID
  WITH MAX_CONCURRENCY = 0;      //设置单节点并发度为0，不允许匹配到的SQL执行
```

在能拿到模版ID（在SQL日志、explain命令、会话中）的情况下，我们还推荐使用更精准的基于模版ID的限流。
也可使用[实例会话页面](https://help.aliyun.com/document_detail/314648.html)里“SQL限流”功能，如下操作：
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/220052/1631947403476-982a30ac-fde1-4bb2-8695-fb1626e8d988.png#clientId=uf4700260-96d6-4&from=paste&height=696&id=u7371257d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=696&originWidth=872&originalType=binary&ratio=1&size=126091&status=done&style=none&taskId=u1610b05f-d6ed-4122-9c34-99fc4ba52aa&width=872)

### 案例2: 慢SQL都是同一个SQL

某DBA收到了数据库资源某指标处于高位的报警，该用户查看数据库慢日志和会话均发现有如下的慢SQL：

```sql
+--------+---------------+---------------------+--------------------+---------+------+-------+---------------------------------------------------+-----------------+
| ID     | USER          | HOST                | DB                 | COMMAND | TIME | STATE | INFO                                              | SQL_TEMPLATE_ID |
+--------+---------------+---------------------+--------------------+---------+------+-------+---------------------------------------------------+-----------------+
| 951494 | userxxxxxxxxx | 222.0.0.1:33830     | analy_db           | Query   |   40 |       | select * from bmsql_oorder where o_carrier_id = 2 | 438b00e4        |
| 952468 | userxxxxxxxxx | 222.0.0.1:33517     | analy_db           | Query   |   43 |       | select * from bmsql_oorder where o_carrier_id = 2 | 438b00e4        |
| 953468 | userxxxxxxxxx | 222.0.0.1:33527     | analy_db           | Query   |   43 |       | select * from bmsql_oorder where o_carrier_id = 2 | 438b00e4        |
| 954468 | userxxxxxxxxx | 222.0.0.1:33537     | analy_db           | Query   |   43 |       | select * from bmsql_oorder where o_carrier_id = 2 | 438b00e4        |
| 955468 | userxxxxxxxxx | 222.0.0.1:33547     | analy_db           | Query   |   43 |       | select * from bmsql_oorder where o_carrier_id = 2 | 438b00e4        |
+--------+---------------+---------------------+--------------------+---------+------+-------+---------------------------------------------------+-----------------+
```

分析发现bmsql_oorder中的符合o_carrier_id = 2条件有较多记录，导致了慢SQL，如果使用模版ID限流，则会影响o_carrier_id不是2的SQL语句，如果使用关键字限流则会影响，类似如下的正常SQL：

```sql
select * from bmsql_oorder where o_carrier_id = 2 limit 1;
select * from bmsql_oorder where o_carrier_id = 2 and o_c_id = 1;
```

**​**

**那么如何限流具体的SQL呢？**
答案是：模版ID + 关键字 限流，可以创建如下限流规则：

```sql
CREATE CCL_RULE `KILL_CCL`       //限流规则名称为KILL_CCL
    ON `analy_db`.`*`              //&匹配analy_db下的所有表上执行的SQL
  TO 'userxxxxxxxxx'@'%'         //&匹配来自userxxxxxxxxx用户的SQL
    FOR SELECT                     //&匹配是SELECT类型的SQL语句
  FILTER BY TEMPLATE '438b00e4'  //&匹配模版ID 438b00e4
  FILTER BY KEYWORD('o_carrier_id','2')  //&匹配参数关键字
  WITH MAX_CONCURRENCY = 0;      //设置单节点并发度为0，不允许匹配到的SQL执行
```

如果使用PolarDB-X的CN内核版本为5.4.11以上, 且改SQL不在prepare模式下执行，还可以使用高阶语法进行限流，如下：

```sql
CREATE CCL_RULE `KILL_CCL`       //限流规则名称为KILL_CCL
    ON `analy_db`.`*`              //&匹配analy_db下的所有表上执行的SQL
  TO 'userxxxxxxxxx'@'%'         //&匹配来自userxxxxxxxxx用户的SQL
    FOR SELECT                     //&匹配是SELECT类型的SQL语句
  FILTER BY QUERY 'select * from bmsql_oorder where o_carrier_id = 2'  //&匹配SQL语句
  WITH MAX_CONCURRENCY = 0;      //设置单节点并发度为0，不允许匹配到的SQL执行
```

### 案例3: 慢SQL集包含多个SQL模版

某DBA收到了数据库资源某指标处于高位的报警，该用户查看数据库慢日志和会话均发现有如下的慢SQL：

```sql
+--------+---------------+---------------------+--------------------+---------+------+-------+---------------------------------------------------+-----------------+
| ID     | USER          | HOST                | DB                 | COMMAND | TIME | STATE | INFO                                              | SQL_TEMPLATE_ID |
+--------+---------------+---------------------+--------------------+---------+------+-------+---------------------------------------------------+-----------------+
| 951494 | userxxxxxxxxx | 222.0.0.1:33830     | analy_db           | Query   |   40 |       | select * from bmsql_oorder where o_carrier_id = 2 | 438b00e4        |
| 952468 | userxxxxxxxxx | 222.0.0.1:33517     | analy_db           | Query   |   43 |       | select * from bmsql_order_line where ol_o_id = 2  | 57a572f9        |
| 953468 | userxxxxxxxxx | 222.0.0.1:33527     | analy_db           | Query   |   43 |       | select * from bmsql_new_order where no_w_id = 2   | de6eefdb        |
+--------+---------------+---------------------+--------------------+---------+------+-------+---------------------------------------------------+-----------------+
```

一般这种情况较为复杂，如果一条明显执行效率很高的SQL也成了慢SQL，则不排除是由于网络抖动或者服务节点异常等原因导致运行效率降低从而产生大批量的慢SQL，也可能是由于真正的烂SQL完全耗尽了资源，导致原本正常的SQL也成了慢SQL，需要通过SQL分析具体分析，不在本文的讨论范围内。假设已经确定了需要限流的慢SQL，我们则可以针对每个模版ID创建一个限流规则。但随着限流规则增加，匹配效率会略有降低，当PolarDB-X的CN内核版本为5.4.11以上时，我们推荐使用多模版限流：

```sql
CREATE CCL_RULE `KILL_CCL`       //限流规则名称为KILL_CCL
    ON `analy_db`.`*`              //&匹配analy_db下的所有表上执行的SQL
  TO 'userxxxxxxxxx'@'%'         //&匹配来自userxxxxxxxxx用户的SQL
    FOR SELECT                     //&匹配是SELECT类型的SQL语句
  FILTER BY TEMPLATE('438b00e4','57a572f9','de6eefdb')  //&匹配中其中一个模版ID，则改匹配项算匹配成功
  WITH MAX_CONCURRENCY = 0;      //设置单节点并发度为0，不允许匹配到的SQL执行
```

如果确定会话中的慢SQL是都是需要限流的烂SQL，且PolarDB-X的CN内核版本为5.4.11以上时，可以开启[慢SQL限流](https://help.aliyun.com/document_detail/316676.html#title-ldm-54x-kpr)。也可使用[实例会话页面](https://help.aliyun.com/document_detail/314648.html)里“SQL限流”功能，如下操作：
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/220052/1631947615800-9a3752bc-94e1-4dfe-9902-7dcc4581a17d.png#clientId=uf4700260-96d6-4&from=paste&height=562&id=udc7018e9&margin=%5Bobject%20Object%5D&name=image.png&originHeight=562&originWidth=920&originalType=binary&ratio=1&size=96241&status=done&style=none&taskId=ue743fd93-8cfa-4478-9320-1b222b2c310&width=920)

## DAS上的SQL限流

用户可从“PolarDB-X控制台”->“诊断与优化”，DAS控制台上PolarDB-X页面的里找到实例会话，点击“SQL限流”进入SQL限流页面，目前支持三种限流方式：通过关键词限流、通过SQL模版ID限流、通过执行耗时限流。
![image.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/220052/1635932265275-f59e0070-58f4-42bb-b451-782efff22279.png#clientId=uf15d4343-2b9d-4&from=paste&height=704&id=u1e2caf68&margin=%5Bobject%20Object%5D&name=image.png&originHeight=704&originWidth=958&originalType=binary&ratio=1&size=177576&status=done&style=none&taskId=u45601b80-adac-4c89-a703-055daa4663b&width=958)
DAS上的SQL限流功能，在设计上，将SQL限流视为应急措施，只有当数据库正在发生由于问题SQL导致的性能问题时才采用SQL限流，然后用户及时在根本上解决问题，包括但不限于：停止发送问题SQL、创建索引等。因此，DAS将SQL限流功能放在会话管理页面，用户可在会话中找到正在执行的问题SQL，创建限流规则时需要设置有效时间，到期后，DAS在后端会帮助用户关闭改SQL限流规则。
如下图所示，DAS在获得用户授权后，可向PolarDB-X实例发送安全的SQL语句（禁止DML、DDL语句），当用户在控制台发起创建、查询当前规则、删除规则操作时，DAS会发送符合限流规则语法的SQL语句给PolarDB-X实例，同时更新DAS自己的业务数据库中的限流规则状态。
![DAS&PolarDB-X.jpg](https://intranetproxy.alipay.com/skylark/lark/0/2021/jpeg/220052/1636434061543-393ac1ef-f4ac-40bc-984a-0436205ff114.jpeg#clientId=u5f37bda8-7495-4&from=ui&id=ub74d60f7&margin=%5Bobject%20Object%5D&name=DAS%26PolarDB-X.jpg&originHeight=706&originWidth=1260&originalType=binary&ratio=1&size=77520&status=done&style=none&taskId=ua722150a-90f3-4dd5-a7bc-1c79e749a56)

## 结语

SQL限流为应急措施，可在数据库由于烂SQL导致效率降低的时候，起到快速恢复的作用。对烂SQL进行限流后，用户需要将注意力集中在如何优化烂SQL上，并在合适的时机清空SQL限流规则。



Reference:

