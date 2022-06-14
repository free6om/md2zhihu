*本文由哈佛大学的Yihe Huang在2020年发表于数据库顶会VIDB的一篇关于多核场景下内存事务数据库的优化手段的研究, 获得当年的 VLDB 2020 Best Paper Award。*

# 摘要

&emsp;乐观并发控制（OCC, Optimistic Concurrency Control）作为一种并发控制技术，可以让内存事务数据库没有写冲突(contention)或者冲突较少场景中，表现出很高的性能。然而写冲突会让性能明显下降，近期提出的一些并发控制设计，比如hybrid OCC/locking system和MVCC的变种，声称比最好的OCC系统性能还要好。我们选取了一些并发控制设计，并在不同的冲突程度、workload（包括TPC-C）对它们的性能进行评估，发现是一些与并发控制无关的因素导致到先前研究中OCC系统的性能下降情况。当这些因素都得到很好的实现后，OCC性能并不会在高冲突(high-contention)的TPC-C测试中出现崩溃。同时，我们提出了两种可以让OCC和MVCC在高冲突场景下性能得到大幅提升的技术，commit time updates 和 timestamp splitting，虽然这两种技术不是第一次出现，但是我们把它们成功应用在新的场景下，而且有新发现：当把两者结合的时候，MVCC有3.4倍的TPC-C性能提升，OCC有3.6倍的TPC-C性能提升。

# 1. 介绍

&emsp;多核内存事务的系统的性能优化是当前的一个热门课题，已经有大量的相关研究被发表(*读者可以查看原文获取*)。基于OCC的技术在通过共享内存带宽和避免不必要的内存写入的手段，可以在low-contention的场景获得很好的性能，然后在high-contention的场景中，大量的事务由于冲突而频繁abort，最坏的情况下，会出现***contention collapse***的情况，也就是一堆事务由于不停的冲突导致流量跌零。
&emsp;最近有面向high-contention的workload提出的设计，包括部分悲观并发控制[1]，动态事务重排序[2]，多版本并发控制(MVCC)[3,4], 修改了事务并发控制协议来更好地支持high-contention的事务处理。但是很多评估比较的都是不同的code base，潜在的问题是，不同的代码实现会导致不同的结果，即没有做好控制变量。
&emsp;我们分析了一些内存事务系统，包括Silo, DBx1000, Cicada, ERMIA 和MOCC， 都发现一些不理想的工程实现（我们称之为basic factor）比如抛出异常时需要获得一把全局锁，很大程度上影响了系统在high-contention场景下的性能。
&emsp;为了更好地区别不同的并发控制方法对性能的影响，我们基于一个新的系统STOv2实现了三种并发控制方法：OCC、TiToc、MVCC，保证了良好并且一致的basic factor，然后对三者进行了评估。我们展示了最多64核，benchmark包括 low- 、high-contenion TPC-C, YCSB，benchmarks based on Wikipedia 和 RuBis。在basic factor一定的情况下OCC没有在这些测试中发生崩溃，这个结论和先前的研究有出入，他们认为: OCC在high-contention的场景下会不可用[5], MVCC在各种不同contention程度的场景下都很出色[4]。
&emsp;另外我们提出了两种优化技术：

-   commit time update. 可以减少read-modify-write情况下的冲突，比如由一般的read和write实现的自增操作
-   timestamp splitting. 可以减少两种事务之间的冲突，1.读很少变化的字段的事务；2.写同一行上其他字段的事务

&emsp;这些优化技术需要面向workload针对性地设置参数，但是我们实际运用过程中，也没有花费太大的功夫。这些优化技术对TicToc、MVCC、OCC有帮助。虽然我们不是第一次提出这些技术的人，但是我们的变种是新的，我们首次把他们运用到TicToc和MVCC上。

# 2. 背景

&emsp;STOv2基于STOv1[6]做了一些重构，具有良好的basic factor，并且支持插件式的并发控制方法。我们专注于三种并发控制方法的比较：1. OSTO, OCC的变种；2. TSTO, TicToc OCC变种；3. MSTO，MVCC的变种。
![|center|500x500](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/220052/1642058224008-8566bb8b-1837-4fca-8b56-4a1b3d9f791d.png) 
&emsp;图1描述了STO的系统架构。STO同时使用了hash table和Masstree[7], hash table可用于实现主键索引、不需要有序性的索引，Masstrue用于实现二级索引、需要有序性的索引、支持范围查询。事务都是使用C++程序实现，访问事务型的数据结构，数据结构和STOv2的核心库保证了事务的线性一致性。事务以"one-shot"的方式执行：事务不需要和用户交互，因为在事务开始的时候，事务的参数都已知。我们不支持持久化和网络通信，因为和本文的主要关注点无关。

### 2.1 OSTO

&emsp;OSTO是一种OCC的变种（遵循Silo[8]的OCC协议）。在excution time，事务逻辑生成read set和write set；在commit time通过三个阶段来实现事务serializability：

-   **阶段一** 给write set中的记录加锁，加锁失败则abort事务。选择事务时间戳,标记serialization point
-   **阶段二** 校验read set中的记录，如果有被其他事务修改过或者加锁，则abort事务
-   **阶段三** 给write set中的记录增加新版本，更新写时间戳，释放锁

OSTO在实现的时候，尽量避免memory contention，比如以一种scalable的方式选择事务时间戳，避免持有全局状态字段的引用。使用Read-copy-update(RCU)技术[9]来进行内存回收，可以让事务继续安全访问已经被逻辑删除的记录，可以大大地降低读写冲突。RCU需要一种可以识别什么时候被逻辑删除对象可以被安全释放的技术。
![|center|500x500](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/220052/1642646902048-fd227642-fec4-475e-a18e-5e2371e3f6a6.png) 
图2列出了三种变量。为了标记对象删除，有个事务跑在一个线程th上，将那些对象记录在一个列表上，记录他们的free timestamp为$fts=wts_{th}$。既然对象可能被并发在跑的事务所访问，在那些事务结束前就释放对象是不安全的。当存在某个事务的$rts_{th^{'}}<fts$，既然$gcts<rts_{th^{'}}$, 只有当$rts_{th^{'}}<fts$时才能安全删除对象。有个用于推进epoch的线程，会周期性地增加$wts_g$，然后重新计算$rts_g$和$gcts$。这个仅仅只产生很少的读写竞争，因为这个任务1毫秒跑一次。

### 2.2 MSTO

&emsp;MSTO是一种基于Cicada[4]的MVCC变种，虽然缺少一些Cicada的高级特性和优化。MSTO维护了每个记录的多个版本，事务可以访问这个记录刚刚的版本和当前的版本。只读事务可以无锁化执行，因为MSTO为最近的时间戳维护了一个一致的快照。MVCC让一些在OCC和OSTO中无法提交的读写事务能够提交，比如下图中的事务t1和t2:
![|center|500x500](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/220052/1642658833370-4eef9699-0f13-431d-b358-2e0da95eea5f.png) 
然而这些是以内存开销为代价的，增大了内存分配、垃圾回收、处理器cache的压力，内存原子操作在MSTO中会比在OSTO中使用得更加频繁。
![|center|500x500](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/220052/1642659150259-ec3465c1-5f1a-4c3e-abb3-e5aa0c9f55af.png) 
MSTO使用索引将主键映射到记录，引出一个叫version chain（上面C图所示）的跳转层。一条记录由一个key和可以指向连表中头版本的指针。每个版本包含：写时间戳、读时间戳、一个状态、以及记录的数据和链表中的指针。写时间时间戳代表着事务时间戳，读时间戳表示该记录最近被读取某个已提交事务读取，该事务的时间戳。提交链表：$v_n,...,v_1$, $v_n$是最新版本，那么 $rts_i \geq wts_i, wts_{i+1} \geq rts_i, wts_{i+1} > wts_{i}$
&emsp;当一个事务开始的时候，会获取事务的实行时间戳 $ts_{th}$, 如果是只读事务则$ts_{th}=rts_g$ ($rts_g$时间之前的事务已经全部结束，保证只读事务执行的时候不会有冲突), 否则$ts_{th}=wts_g$
&emsp;MSTO在提交的时候首先获取一个提交时间戳 $ts_{thc}:=wts_{g}++$, 接下来分三个阶段进行：

-   **第一阶段** 插入一个写时间戳为$ts_{thc}$, 状态为PENDING的版本到需要修改的记录的版本链表中
-   **第二阶段** 检查read set，如果在$ts_{thc}$和$ts_{th}$观察到的版本不同，则abort事务；否则，原子性地修改read set记录里的$v.rts:=max\{v.rts,ts_{thc}\}$
-   **第三阶段** 修改PENDING状态为COMMITTED，把老版本加入垃圾回收的队列中。

如果事务失败，则PENDING状态将会被修改为ABORTED。本提交协议只适用于读/写事务，只读事务永远都可以得到提交。

&emsp;MSTO结合了一个重要的Cicada的优化，称为inlined version, 某个版本可以被存储在记录上，减少内存访问地址的跳转，在更新频繁的情况下，可以缓解cache的压力。MSTO在inlined的记录是空的或者被垃圾回收了，会对其进行填充。

### 2.3 TSTO

TSTO是OSTO的一个变种，它使用TicToc[1] 而不是简单的OCC技术。TicTic和MVCC类似，每个记录有一个read timestamp和一个write timestamp，但只保留最新的版本，它会基于read set和write set动态地计算提交时间戳，这样允许更灵活的事务调度。

# 3. 实验配置

&emsp; 我们在一个一台带有两颗Intel Xeon E5-2686 v4 CPU(一颗为16核32线程)，内存为256GB的机器上做实验。每个case做做5次，取中位数，最大值和最小值用作error bar。实验中，被abort的事务会在同一个线程中发起重试直到提交成功。Workload上我们选择，YCSB A/B ，TPCC，通过参数修改workload在high-contention和low-contention的workload。

# 4. 基本要素

主存事务处理系统出了收到并发控制策略的影响，还收到一些基本要素很大的影响，比如内存分配、索引类型、异常抛出策略。下图展示了，在OSTO上的测试结果，其中粗线是基准线，是一个基本要素都实现良好的系统的测试结果
![|center|500x500](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/220052/1642942751392-fa99a26a-968f-4dfb-89ca-1606d52754ca.png) 
在每个case中单独修改一个factor，该factor实现选自之前的系统，可以看出high-contention的TPC-C的workload中，四个基本要素能有20%的性能影响，有两个要素导致崩溃。在TSTO和MSTO也有类似的结果，其中内存分配器对MSTO性能影响更大（多版本消耗占用更多内存）。

下图所示，通过实验和代码分析，比较了8个系统的7个要素（STOv2为基准）：
![|center|800x500](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/220052/1642944229599-0a5a26d3-aae7-47d8-b73f-d8993ba76c61.png)

# 5. 并发控制评估

&emsp; 针对发现的对性能有很大影响的一些基本要素，我们都在STOv2进行了很好地实现，通过修改它的并发控制策略，来评估不同并发控制策略的表现。
&emsp; 先前的研究工作表明，在high-contention的TPC-C负载中OCC的性能发生了崩溃，而在本次性能评估中并没有发现这个现象。在high-contention的TPC-C负载中，即使在64线程下，OSTO的吞吐量大约是MSTO的60%。没有任何一个系统是可扩展的，也有没有崩溃的。在low-contention的情况下，OSTO的吞吐量大约是MSTO的2倍。评估结果如下图所示：
![|center|500x500](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/220052/1642945359457-ec5b8f39-2f76-4fcf-924c-f43449775062.png)

# 6. 在high-contention下的优化

&emsp;我们的commit-time update(CU) 和 timestamp splitting(TS)优化技术可以减少很多来自workload的冲突。在我们的性能评估中，在各个系统，各种workload下，这两个优化都能让性能提升，有的case显示有大幅提升。他们还展示了协同作用，在一些workload下，TS使CU更加高效。由于这两个优化技术需要根据workload来调整，因此需要事务的编程者付出一些努力。CU和TS也减少了一些并发控制协议无法免去的冲突，比单独只有并发控制协议的情况下的性能有较大提升。

## 6.1 Commit-time updates

&emsp;OCC和MVCC系统中有很大一部分读写集合是read-modify-write操作，比如自增操作，读取一个旧的值，然后写入一个新的值。然而这么操作会导致不必要的冲突，我们通过盲写（写集合中标记下，表示事务提交的时候需要被自增一下），来避免记录进入事务的读集合，从而可以减少很多冲突。
&emsp;STOv2中的commit-time update特性将特定的read-modify-write操作表示成各种盲写操作。updates表示对一个记录类型的操作，写集合中，可以是一个值，也可以是一个updater。每个updater都表示基于一条记录和参数所进行的操作，不能访问其他记录，而一个事务可以产生多个updater。如过下图所示，是一个操作stock记录的updater:
![|center|500x500](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/220052/1643004153880-c73352d0-37a6-4318-940c-d7a1dbdd67d2.png) 
&emsp;OCC和TicToc的updater可以在commit时候的第三个阶段，也就是相关记录已经被锁住的时候，被调用。而在MVCC中，updater被加入到版本链表中作为独立的实体，被称为delta version。当版本链表中存在delta version的时候，进行读取操作的时候，需要先进行flattening操作，得到最终的值。

## 6.2 Timestamp splitting

&emsp;很多数据库记录有多个访问模式不同的片段组成。比如一条记录有很多列，有的列被频繁访问，而有的则不怎么被访问，或者他们具有不同的访问模式。Schema变更中的row splitting和vertical partitioning使用这种模式来减少数据库的IO开销，被频繁访问的字段会被缓存到内存里。timestamp splitting优化使用这些模式可以避免很多冲突。
timestamp splitting将一条记录拆分成几个子集，每个子集一个时间戳。当修改记录中的列的时候，只修改覆盖到了这些列的时间戳。如下图所示，频繁和不频繁访问的列都有自己的时间戳：
![|center|500x500](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/220052/1643005908884-e7eb0354-fb81-4526-83c5-d08da9b3cb3a.png) 
&emsp;OSTO和MSTO通过上图的方式来实现timestamp splitting，然而MSTO则选用了更重量级的实现方式：将不同访问模式的列拆分到不同的表中，我们之后会想办法通过更轻量级的方式在MSTO上实现timestamp splitting。

# 7. 优化点的评估

&emsp;我们进行了在TPC-C, YCSB, Wikipedia 和 RUBiS的流量负载下对优化后的STOv2进行了评估。

## 7.1 一起使用的效果

&emsp;如下图所示，在high-contention的几种workload下，commit-time updates(CU)和timestamp splitting(TS)的结合都能让三种并发控制技术性能都有提升。在high-contention的TPC-C负载下，CU+TS的结合对三种CC技术可以有2到3.9倍的提升。在low-contention的TPC-C情况下，CU+TS对OSTO和TSTO产生了大约10%的性能开销，而对MSTO产生了约30%的性能开销，这是由于对长的delta version连表具有很大的垃圾回收的开销，可以在low-contention的情况下关闭MVCC系统的commit-time update(CU)优化。
![|center|800x500](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/220052/1643008691623-8db6bf2f-0140-4662-b681-6077aff91b66.png)

## 7.2 分开使用的效果

下图，展示了两个优化点分别使用的效果：
![|center|800x500](https://intranetproxy.alipay.com/skylark/lark/0/2022/png/220052/1643009841943-b15eef76-e30a-4e4a-9f4c-680f1c584401.png) 
&emsp;可以发现的case下单独使用CU或者TS, 性能会下降，而同时使用CU和TS的时候又能使性能产生数倍的提升。那是因为很多被频繁更新的字段，只有那些不被频繁更新的字段使用了不同的时间戳的时候，才能通过使用CU来更新。

# 8. 结论

&emsp;我们研究了三种用于提升主存事务处理系统在high-contention情况下的吞吐量的方法：基本要素的提升、并发控制算法、high-contention优化。不好的基本要素的实现，系统会在high-contention的情况下发生崩溃，流量跌0。当基本要素实现良好的时候，两项针对high-contention场景的优化：commit-time updates和timestamp splitting的优化提升比修改并发控制算法更有效果。CU和TS同时使用可以在TPC-C场景下使基本的并发控制算法提升到3.6倍，然而不同的未经优化的并发控制算法之间的差距也最多只有2倍。
&emsp;我们当前的方案还不是最优的，不过受到CU+TS的优化点的启发，未来有可能出现一种可自适应worklad的并发控制算法会出现。提升high-contention场景下的主存事务的性能，最好的方式是减少冲突，虽然我们当前的方法需要人工参与，但希望未来的研究可以让CU和TS可以自动运行。
&emsp;最后，我们被OCC无论在low-contention或者high-contention的场景下都有很好的性能所震惊，即使MVCC以及其他并发控制技术可能在某些workload下有绝对的性能优势。

# 引用

[1] T. Wang and H. Kimura. Mostly-optimistic concurrency control for highly contended dynamic workloads on a
thousand cores. PVLDB, 10(2):49–60, 2016.
[2] X. Yu, A. Pavlo, D. Sanchez, and S. Devadas. TicToc: Time traveling optimistic concurrency control. In Proceedings of the 2016 International Conference on Management of Data, SIGMOD ’16, pages 1629–1642. ACM, 2016.
[3] K. Kim, T. Wang, R. Johnson, and I. Pandis. ERMIA: Fast memory-optimized database system for heterogeneous workloads. In Proceedings of the 2016 International Conference on Management of Data, SIGMOD’16, pages 1675–1687. ACM, 2016.
[4] H. Lim, M. Kaminsky, and D. G. Andersen. Cicada: Dependably fast multi-core in-memory transactions. In Proceedings of the 2017 International Conference on Management of Data, SIGMOD ’17, pages 21–35. ACM, 2017.
[5] J. M. Faleiro and D. J. Abadi. Rethinking serializablemultiversion concurrency control. PVLDB, 8(11):1190–1201,2015.
[6] N. Herman, J. P. Inala, Y. Huang, L. Tsai, E. Kohler, B. Liskov, and L. Shrira. Type-aware transactions for faster concurrent code. In Proceedings of the 11th European
[7] Y. Mao, E. Kohler, and R. T. Morris. Cache craftiness for fast multicore key-value storage. In Proceedings of the 7th European Conference on Computer Systems, EuroSys ’12, pages 183–196. ACM, 2012.
[8] S. Tu, W. Zheng, E. Kohler, B. Liskov, and S. Madden. Speedy transactions in multicore in-memory databases. In Proceedings of the 24th ACM Symposium on Operating Systems Principles, SOSP ’13, pages 18–32. ACM, 2013.
[9]P. E. McKenney and S. Boyd-Wickizer. RCU usage in the Linux kernel: One decade later. Technical report, 2012.



Reference:

