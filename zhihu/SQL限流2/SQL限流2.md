## 介绍

我们在PolarDB-X SQL限流的基础版本上做了进一步的性能优化和功能扩展，本文将对此进行介绍并进行技术细节上的剖析。如果读者对PolarDB-X SQL限流的基础背景知识感兴趣，可先阅读之前在专栏发的[PolarDB-X SQL限流](https://zhuanlan.zhihu.com/p/345227459?utm_source=wechat_timeline&amp;utm_medium=social&amp;utm_oi=61797981749248&amp;utm_content=first)。接下来，我们首先以SQL限流初版上线后的遇到的一个典型案例展开，对此进行分析，让读者对SQL限流有个很好的体感，随之引出新特性的介绍和剖析。
​

## 案例

随着业务的快速发展，数据规模急剧膨胀，一些数据库用户最终会走上将业务迁移到分布式数据库的道路。阿里云凭借自己在云计算领域的先发优势以及雄厚的研发实力，获得了一众客户的信任，而PolarDB-X作为阿里云数据库里的一个明星分布式数据库，出现在客户们的视野中。
迁移是较为复杂的过程（要求：SQL语法兼容、性能不下降等），需要有完备的方案来尽可能减少停机时间（平滑迁移），用户在迁移过程中难免会遇到一些小困难，比如在源数据库中建好的索引，而在PolarDB-X中漏建了该索引，导致需要该索引的SQL语句执行性能出现下降，甚至拖垮整个数据库的运行效率，比如下图所示，全表遍历导致DN (Data Node, 存储节点) CPU满：
![image.png](https://cdn.jsdelivr.net/gh/fuyufjh/md2zhihu@_md2zhihu_Downloads_40b3de68/zhihu/SQL限流2/1629450676597-6fadee43-ed07-4cca-8aa2-0db6c25ce281.png)
    使用SQL诊断工具和show full processlist查看，均发现有条执行时间很长的慢SQL，经用户确认，可以对该SQL的进行限流，我们对该SQL模版发起了限制并发度的限流来缓解CPU满的情况，发现CPU不再100%后再对该SQL的执行计划进行分析，发现是用户没建索引导致了全表遍历从而大量消耗资源，建议用户创建索引，一段时间后（有一定数据量，创建索引需要时间）该SQL执行效率提高，CPU利用率大幅下降，就此迁移计划圆满完成，客户很满意。
![image.png](https://cdn.jsdelivr.net/gh/fuyufjh/md2zhihu@_md2zhihu_Downloads_40b3de68/zhihu/SQL限流2/1629450664688-6813213e-9154-4956-8293-f689efc8bc4c.png)

### 分析

```
总体来讲，这是一个比较成功的应用案例，但本着精益求精的精神，我们仍然对有待提升的方面进行了分析。
```

#### 分析点 1: 能否进一步减少对用户业务的影响？

```
这次使用限流为限制并发度，超出并发度的SQL语句会执行失败，客户反馈有上游业务反馈有偶发的查询报错。本案例中，这条SQL为一条不是很重要的SQL，因此偶发的报错其实并不会给客户造成多大的损失。而如果这是一条比较重要的SQL，不允许出现报错，则我们可以设置限流语法with_options中的WAIT_QUEUE_SIZE来建立一个等待队列，起到削峰填谷的作用。然而，PolarDB-X在接收到SQL语句后，会从线程池中为其执行任务分配一条线程，SQL限流初版通过让线程等待（Thread.wait，不归还线程给线程池）的方式来实现执行任务的等待，因此执行任务进入等待队列后仍然占用线程池中一条线程，当等待队列设置过长时有耗尽线程池资源的风险，并有可能引发死锁问题。
```

![ccl_wait_queue.png](https://cdn.jsdelivr.net/gh/fuyufjh/md2zhihu@_md2zhihu_Downloads_40b3de68/zhihu/SQL限流2/1629203846792-e7921430-2e50-4d8c-878a-8ed8794c043a.png)

#### 分析点 2: 能否更加直观的使用和展示限流规则？

在本案例中，我们使用一个烂SQL的模版ID（可从SQL日志和show full processlist中获得）的作为匹配条件。比如创建一个限流规则，需要输入如下创建语句：

```sql
CREATE CCL_RULE `DEMO_CCL_RULE` on *.* to 'testuser'@'%' for SELECT FILTER BY TEMPLATE('aaeecc') with MAX_CONCURRENCY = 1
```

通过 show ccl_rules查看已创建的规则，展示限流规则中，我们可以看这个模版ID。然而，如果过段时间再去看这个限流规则，我们没法非常直观地联想到，这个规则到底限制的是什么样子的SQL？
​

#### 分析点 3: 能否一键限流？

```
PolarDB-X为SQL限流设计了简单明确的语法，但是对用户来说仍然存在一定的上手门槛，能否在系统由于烂SQL出现卡慢的时候，用户只需要简单输入一个一条非常简单的命令就能实现有效限流，并且无需关心烂SQL有哪些具体的特征（数据库、表、DB账号、模版ID等）？
接下来，我们将一一解答上面分析中所遇到的问题。
```

​

## 不占用线程的等待队列

在SQL限流的第一版中，我们通过Thread.Wait的方式实现了线程的等待，然后这种方式会让不在执行状态的查询占用一些不必要的资源（比如：线程资源、保存执行上下文的内存资源等），从监控上可以看到正在等待的查询，仍然会计算入“活跃会话数”的指标里。如果将等待队列设置过大，存在耗尽线程资源和出现死锁的风险。我们对此进行了优化，通过SQL的重调度实现无需要占用执行所需的中间资源的等待队列。
架构图如下所示：
![no_thread_wait_queue.png](https://cdn.jsdelivr.net/gh/fuyufjh/md2zhihu@_md2zhihu_Downloads_40b3de68/zhihu/SQL限流2/1629203902795-3859319b-04fb-437a-b731-fd78b8c2de46.png)
这个架构图展示了一条普通SQL在PolarDB-X CN节点里执行历程，我们的在限流匹配模块（CCL Match）放在查询优化器之后执行，如果被匹配到并需要进入等待队列，做如下步骤：

1.  打包Reschedule时所需要的参数和Connection对象，将其放入属于该限流规则的等待队列。
1.  抛出CclReschedulableException异常，让上层模块知晓这个请求之后会重新调度，因此不给客户端返回错误信息，不打本轮执行的SQL日志。

重新调度的思路说简单也简单，如果一个函数的结果只取决于它的输入，那么我们需要保留这个函数的入参即可实现重新调度，比如我们想计算 z = x + y 的值，A和B客户端并发提交计算任务，要求任意时刻只能有一个这样的任务在执行，任务等待队列实现的伪代码可以如下：

```code

//A和B 并发提交计算任务
void compute(ResultStream resultStream,int x, int y){
    try{
    	int result = sum( x , y );
      resultStream.write(result);
      //尝试唤醒等待队列里的任务，取出保留x、y参数和返回流对象，重新提交执行任务
      tryCclReschedule();
    }catch(Exception exp){
      if(exp instanceof CclReschedulableException)
    	 {
    			//保留x、y参数和返回流对象，操作进入等待队列
    		}
    		else{
    		 //给客户端返回异常
    		 resultStream.WriteError();
    		}
    }
}

//执行sum计算的函数
int sum(int x, int y ){
   //判断并发度，超过1后跑出需要重新调度的异常
   if (concurrency > 1){
     //实际并发度的记录需要通过计数器来实现
    	throw new CclReschedulableException();
   }
   return x + y;
}
```

另外，架构图中的"wakeup ccl waiting sql"模块：为占用并发度的SQL在执行完后尝试唤醒队列中的查询。“Timeout Checker”模块检测等待队列中任务，抛弃超时任务并给客户端返回限流等待超时的报错。

##### 等待队列是如何实现挑选出超时任务并进行处理？

我们采用双端队列，每个提交到队列的查询任务都有有个系统时间戳，用于计算等待时间，如下图所示：
![image-20210817164906579.png](https://cdn.jsdelivr.net/gh/fuyufjh/md2zhihu@_md2zhihu_Downloads_40b3de68/zhihu/SQL限流2/1629204032857-cf130f28-5bbb-4b8d-81cf-6a318576f0f7.png)
可以发现越接近队列右侧头部的查询任务的等待时间越长，因此超时检测，只需要从头部开始检测即可，如果头部第一个查询任务没有超时，说明都没有超时，如果超时则丢弃，继续检测头部直到没有超时的任务的为止。值得注意的是，唤醒操作和超时任务访问的同一个队列头部，需要防止出现丢弃被唤醒任务的情况（处理细节不再解释）。

## 直接输入SQL语句进行限流

按PolarDB-X SQL限流初版的功能，如果我们想要限制某个查询，则需要知道该查询的关键词或者模版ID用于该查询的特征匹配，缺点很明显，在show ccl_rules查看限流规则的时候无法很显然地知道到底限的是什么SQL，为解决这个问题，我们开发了基于SQL语句的限流规则创建方式，在语法上以Filter By Query的方式提供，是对Filter By Keyword和Filter By Template的补充。

#### 例子

我们希望限制的具体SQL如下(其模版ID假设为 aaaccc)：

```sql
SELECT * FROM `bad_sql_table` WHERE `name` = 'bad_sql' and `status` = 1;
```

按初版我们可以使用如下语法（模版ID + 参数匹配）：

```sql
CREATE CCL_RULE testrule ON *.* to 'testuser'@'%' 
   FOR SELECT 
   FILTER BY TEMPLATE 'aaaccc' 
   FILTER BY KEYWORD('name','bad_sql','status','1')
   WITH MAX_CONCURRENCY = 1
```

而在新版上我们可以使用如下语法：

```sql
CREATE CCL_RULE testrule ON *.* to 'testuser'@'%' 
   FOR SELECT 
   FILTER BY QUERY "SELECT * FROM `bad_sql_table` WHERE `name` = 'bad_sql' and `status` = 1"
   WITH MAX_CONCURRENCY = 1
```

相比上一种明显直观很多。另外还支持使用 ? 表示任意匹配, 比如SQL模版匹配：

```sql
CREATE CCL_RULE testrule ON *.* to 'testuser'@'%' 
   FOR SELECT 
   FILTER BY QUERY "SELECT * FROM `bad_sql_table` WHERE `name` = ? and `status` = ?"
   WITH MAX_CONCURRENCY = 1;
```

或者, 部分参数匹配：

```sql
CREATE CCL_RULE testrule ON *.* to 'testuser'@'%' 
   FOR SELECT 
   FILTER BY QUERY "SELECT * FROM `bad_sql_table` WHERE `name` = 'bad_sql' and `status` = ?"
   WITH MAX_CONCURRENCY = 1;
```

#### 实现方式

根据Filter By Query的中的SQL语句，我们可以提取出它的模版ID，参数（位置和值）。
比如，比如从以上限流的具体SQL我们可以提取类似以下的数据结构：

```code
{
  "ParamterizedSql" : "SELECT * FROM `bad_sql_table` WHERE `name` = ? and `status` = ?"
  "TemplateId": "aaaccc"
  "Params": {0:"bad_sql", 1: 1}
}
```

TemplateId是ParamterizedSql的hash值。最后的匹配结果可以表示为： match templateId && match params。
​

注：由于Prepare模式下执行的SQL的模版ID不是参数化后的SQL的hash值，因此Filter By Query对Prepare模式下执行的SQL不适用。
​

## 限流触发器&一键限流

本节简单介绍基于运行时信息的限流触发器，利用这个触发器，我们实现了一键限流。
​

### 限流触发器

为了SQL限流更加智能，我们开发了限流触发器，基础的SQL限流可以认为是使用SQL的关键词流来归类SQL，而限流触发器可以实现根据运行时信息来归类SQL，比如运行时间。限流触发器的运行框架如下：
![ccl_trigger.png](https://cdn.jsdelivr.net/gh/fuyufjh/md2zhihu@_md2zhihu_Downloads_40b3de68/zhihu/SQL限流2/1629204066828-a0a8cb72-acee-42aa-99f0-e2050751e9db.png)
当一条SQL执行的时候，我们会统计一些关于这次执行的一些运行时信息，做profiling的时候会非常有用，其中包括执行时间，我们可在SQL执行完成的地方埋点，如果符合触发器的条件，加入一个队列中，排队创建限流规则。
​

其中，有个定时任务会周期性地去拉取队列中的限流规则创建任务，一次性拉取出所有的限流规则创建任务，该定时任务的操作包括：

-   **拉取限流规则创建任务**。一次性拉取完队列中所有的限流规则创建任务，开始处理这批任务。
-   **校验限流触发器。**主要检查限流触发器还是否有效, 如果已经被用户删除或者限流规则数量达到上限，则不需要再执行这个限流规则的创建任务。
-   **创建限流任务。**根据指定的信息创建限流规则，使用这条SQL的schemaName并通过filter by query的方式进行创建，限流规则with选项采用限流触发器里设定的。
-   **合并限流任务。**多个基于templateId的规则，会合并成单限流规则多templateId的形式。
-   **结束任务**

### 一键限流

一键限流旨在降低用户使用SQL限流的门槛，如果确定是慢SQL导致的数据库性能问题（很常见），便可以通过极其简单命令开启SQL限流。

#### 开启一键限流

语法：

```sql
SLOW_SQL_CCL GO [ SQL_TYPE [MAX_CONCURRENCY] [SLOW_SQL_TIME] [MAX_CCL_RULE]]
```

-   SQL_TYPE取值：'ALL'，'SELECT'， 'UPDATE'， 'INSERT'，默认为'SELECT'
-   MAX_CONCURRENCY默认值为计算节点CPU核数的一半
-   SLOW_SQL_TIME默认值为系统参数SLOW_SQL_TIME的值
-   MAX_CCL_RULE的默认值为1000

**运行步骤：**

1.  遍历整个实例的session，识别出该语句类型慢SQL的TemlateId
1.  创建针对慢SQL的限流触发器
1.  传递慢SQL的TemplateId给限流触发器，由限流触发器创建限流规则
1.  Kill所有该语句类型的慢TemplateId查询

一般情况下，用户只需要输入 SLOW_SQL_CCL GO即可实现一键限流。

#### 一键关闭限流

语法：

```sql
SLOW_SQL_CCL BACK
```

删除由SLOW_SQL_CCL创建的限流触发器。

#### 查看限流情况

当下发限流命令时，我们可以通过如下命令，来评估这个命令产生的效果，即查看哪类SQL可能会被限流。
语法：

```sql
SLOW_SQL_CCL SHOW
```

实现方式：plan_cache和ccl_rules里以模版ID作为join key的一次inner join
如下图所示，显示了select sleep(10)会被限流，且该SQL模版在每个计算节点的最大并发度为2。
![slow_sql_ccl_show.png](https://cdn.jsdelivr.net/gh/fuyufjh/md2zhihu@_md2zhihu_Downloads_40b3de68/zhihu/SQL限流2/1629204090815-d5067283-8d72-4b57-9091-eba06ce57547.png)

## 总结与展望

针对PolarDB-X SQL限流初版，我们在最新版本上进行了功能、性能、体验上的升级，包括：1.增加Filter By Query的能力；2.支持一个限流规则设置多个模版ID；3.支持慢SQL一键限流；4.等待队列让出线程资源。当前PolarDB-X正在全面对接阿里云 ''数据库自治服务 DAS"，PolarDB-X限流能力将会以白屏化的方式提供给用户，经一步提升用户体验，降低使用门槛。



Reference:

