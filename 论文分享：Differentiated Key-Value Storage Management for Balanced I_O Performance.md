# 导语

LSM-Tree 作为一种为写优化的数据结构，近年来随着 LevelDB 和 RocksDB 的广泛使用逐渐走入大家的视线。在使用 LSM-Tree 时，较大的写放大会对系统整体的写入性能造成影响；而 KV 分离技术虽然能够降低写放大，但却会影响查询性能。

本文将介绍发表在 ATC 2021 上的 《Differentiated Key-Value Storage Management for Balanced I/O Performance》[1]，其在现有 LSM-Tree KV 分离技术做出一系列改进，通过对 Value 排序、细粒度 KV 分离等方式，使得其能够在不同的工作负载下都能有一个良好的性能。

# 背景

下图展示了 LSM-Tree 的基本结构。LSM-Tree 在磁盘上是一个分层存储的结构，越往下层数据量越大。当我们将一个 KV 对写入 LSM-Tree 时，我们的写请求首先缓存至内存的 MemTable，MemTable 写满后再 Flush 到 L0 的一个 SSTable 文件中。而写入 SSTable 后的 KV 对则通过 Compaction 操作逐层向下移动。由于 Flush 和 Compaction 都是顺序访问磁盘的，因此其有一个较好的写性能。

![截屏2021-12-01 下午9.46.55.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/25656933/1638366445599-78cc6f52-c09a-48bb-84db-46536d2717aa.png)

在 LSM-Tree 中，写放大指的是写入一个 KV 对的引起的磁盘 I/O 大小和实际 KV 对的大小之比，那么在磁盘带宽有限的情况下，写放大越大，实际数据写入的吞吐量也就越小。在 LSM-Tree 中，影响写放大的操作主要是 Compaction，因为一次 Compaction 需要将下层 SSTable 中的 KV 对读取出，归并排序后再写回同一层。假设层间大小之比为 $T$ 的 LSM-Tree，一个 KV 对平均在一层中经历 $T$ 次 Compaction 后才会移动到下一层，因此 KV 对每经过一层写放大就会增加 $T$。

为了减少写放大，目前学术界的优化方案可以大致分为两类：

-   放松每一层 KV 对的顺序约束。之所以 Comapaction 会产生写放大，就是因为为了保证每一层的 KV 对都是有序的，每次将上层的 SSTable 向下层移动的时候都需要重新进行一次归并排序。因此在第一种做法放松每一层 SSTable 都要有序的这一限制，允许每一层存在多个 Sorted Group，这样就不会在每次 Compaction 的时候都触发与下层文件的归并排序。这种做法的缺点是牺牲了查询的性能，因为现在在查询时每一层都要进行多次查找。目前一些相关的文章有 PebblesDB[2]、Dostoevsky[3] 等。

    ![截屏2021-12-01 下午9.47.04.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/25656933/1638366465310-28970b6d-5d3b-468f-a403-cd58c8e09957.png)

-   KV 分离。在一些使用场景下，Key 和 Value 的大小相差较大，于是我们将 Value 单独存储在一个追加写的 vLog 中，只在 LSM-Tree 存储 Key 和指向 Value 的指针，这样进行 Compaction 的时候每次都需要重新读取和写入的数据量就会小很多。但这种做法同样带来了一些新的问题：

    1.  由于 Key 和 Value 是分开存储的，所以每次读取一个 KV 对的时候我们需要多进行一次 I/O 来获得 Value，影响查询性能。此外由于 Value 是按写入顺序而不是 Key 的顺序进行存储的，因此在范围查询时会导致大量的随机 I/O，范围查询的性能较差。
    1.  由于在 LSM-Tree 中删除一个 Key 时不能直接删除 vLog 中的 Value ，因此我们需要一个额外的 GC 过程来定期回收被删除的 vLog 中的无效 value 占用的空间。我们以 WiscKey 为例，其做法是在 GC 的时候每次选择最旧的一个 vLog 文件，然后把里面的无效 Value 丢掉，把依然有效的 Value 重新 Append 到 vLog 尾部。这一过程需要重新查询 LSM-Tree 并更新相应的 Value 指针，相比于 LSM-Tree 的删除代价更高。

    目前一些相关的文章有 WiscKey[4]、HashKV[5] 等。

    ![截屏2021-12-01 下午9.47.08.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/25656933/1638366495179-28887df6-7452-40ef-8072-a6b2299011aa.png)

# 出发点

为了说明这两类优化方案的效果，论文中进行了一系列实验，用 RocksDB 和放松每一层 KV 对的顺序约束的 PebblesDB、KV 分离的 Titan 进行对比：

-   在写入性能方面，实验中 Load 100GB 随机数据，然后观察写放大和写吞吐。可以看出，使用 KV 分离的 Titan 写放大最小，写吞吐最高；而 RocksDB 的写放大最大，写吞吐最低。可以看出，这两类优化均在一定程度上提升了 LSM-Tree 的写性能。

    ![截屏2021-12-01 下午9.55.02.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/25656933/1638366913040-f0276df5-b91a-49e2-a3a8-7a8d562a63aa.png)

-   在查询性能方面，左图展示的是点查询的吞吐，可以看到比较有意思的一点是 Titan 虽然用了 KV 分离，需要多一次 I/O 来读取 Value，但是其点查询性能反而是最好的，其原因是 KV 分离之后 LSM-Tree 本身较小。

    右图展示的则是范围查询的延迟，从图中可以看出 RocksDB 的表现是最好的，特别是在 Value 大小较小的时候。另一方面，通过对延迟的分解，可以发现范围查询时主要的时间开销在于迭代器的迭代部分。特别是对于 KV 分离的 Titan，其迭代的时候会产生大量的随机 I/O，并且虽然每次都读取 4K 的内容至内存，但是只有一部分是有用的；而其它两种做法，由于是按 Key 的顺序进行存储的，所以可能一次 I/O 就读取了多个需要的 KV 对至内存中。

    ![截屏2021-12-01 下午9.55.35.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/25656933/1638366948164-10e1798e-5a78-492b-8be5-f317f10ef7b8.png)

这两组实验说明了现有的写放大优化技术虽然确实优化了写性能，但在一定程度上牺牲了查询性能，特别是范围查询，因此论文中提出了一种新设计，即 DiffKV，其通过对 Value 排序、细粒度 KV 分离等方式，能够在无论是写入，还是点查询、范围查询时都能取得一个较好的性能。

# 设计

DiffKV 依然采用了 KV 分离的设计，其核心思想是 Value 将其存储 vTree 中，vTree 采用了放松顺序约束的方法进行管理。具体来说，vTree 的每一层由若干个 Sorted Group 组成，一个 Sorted Group 中的 Value 是按 Key 的顺序存储在 vTable 中。一开始在 Flush 的时候，生成一个 SSTable 的时候也会生成一个相应的 vTable；之后上层 vTable 中 Value 会通过 Merge 的方式往下层移动。

![截屏2021-12-01 下午9.57.10.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/25656933/1638367038654-f847bb5e-d157-4e86-aa46-9a400ca40863.png)

## Compaction-triggered Merge

接下来我们看一下 Merge 操作是怎么进行的。首先，一个 SSTable 中的 Key 所对应的 Value 可能在不同的 vTable 中，因此如果 Merge 操作是独立进行的话，我们依然需要读取 LSM-Tree 来判断哪些 Value 是有效的，并且更新这些 Value 在 LSM-Tree 中的指针，代价较高。因此 DiffKV 采用了 Compaction-triggered Merge，其基本思想是在 LSM-Tree 进行 Compaction 的时候顺便进行 vTree 的 Merge 操作。具体做法是，在 Compaction 的时候，将这次 Compaction 所涉及到的 Value 全部读取出来，并 Append 到下一层，形成一个新的 Sorted Group。由于 Compaction 操作本来就要在 LSM-Tree 中重新读取和写入 KV 对，所以 Merge 操作就不需要额外的查询，代价较低。由于 Merge 之后的 vTable 可能仍然含有有效的 Value ，所以这些 vTable 的空间之后由 GC 进行回收，而不是直接删除。

![截屏2021-12-01 下午9.57.31.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/25656933/1638367085569-6e1ae6db-89eb-405d-8991-3097624b3660.png)

但这样做也有两个问题：

1.  如果每次 Compaction 都触发一次 Merge 的话，那么 Merge 的次数依然很高，总体代价依然较大；
1.  其次由于 Merge 是 Append Only 的，那么在运行一段时间后 vTree 的每一层存在很多的 Sorted Group，影响查询性能。

针对第一个问题，文中提出了 Lazy Merge。因为 LSM 中的大部分数据都是保存在最后几层，这也就意味着范围查询的时候大多数值实际上是从 vTree 的最后两层进行扫描得到的，所以主要是最后两层的 Value 的有序程度决定了范围查询的性能，而上层 Value 的有序程度对范围查询的性能影响很小。因此我们选择只在最后两层进行 vTree 的 Merge 操作。

![截屏2021-12-01 下午9.58.17.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/25656933/1638367109324-d8c539ee-843a-4560-af5b-8537c3c2d4b9.png)

针对第二个问题，文中提出了 Scan-optimized Merge。DiffKV 在 Merge 的时候会检查同一层中范围重叠的 vTable 数量，过多时进行将这些 vTable 打上标记，以供之后 GC 时进行回收。

![截屏2021-12-01 下午9.58.35.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/25656933/1638367124079-347e8073-a859-407b-a0bb-44858db9e106.png)

## GC

为了进行 GC，DiffKV 在内存中保存了所有 vTable 中无效 Value 的数量，如果一个 vTable 中的无效 Value 数量超过阈值，我们对 vTable 进行标记。那么在之后的 Compaction 的过程中，如果发现一个如果 Value 处于被标记的 vTable 中，我们就会将其写入新生成的 vTable 中，而原来的 vTable 中的 Value 就会标记为无效。如果一个 vTable 中所有的 Value 都是无效的，系统就可以安全地将其删除。简单来说，当一次 Merge 时，在最简单的情况下只有上层 vTable 的 Value 会写入新生成的 vTable 中，而标记使得处于下层的 Value 也可能会被重写，Append 至一个新的 Sorted Group 中。

## Fine-grained KV Separation

在提出了 vTree 之后，文中进一步提出了细粒度的 KV 分离，其基本思想是在 SSD 环境下，当 Value 大小足够大时，使用最简单的 vLog 也能有较好的范围查询性能，并且还能减少写一次 WAL。因此本文将 Value 大小分为大、中、小三种，大 Value 写入 vLog 中，中 Value 写入 vTree 中，而小 Value 直接和 Key 一起写入 LSM-Tree 中。

![截屏2021-12-01 下午10.02.24.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/25656933/1638367354797-c55030da-8b2f-4c5f-93fa-0d14731f8c92.png)

vLog 同样需要进行 GC，因此 DiffKV 在内存中监控了每个 vLog 文件的无效数据比例。GC 触发的时候，DiffKV 选择无效 KV 占比最大的文件进行回收空间。并且 DiffKV 还实现了一个简单的冷热分离策略，其把 MemTable Flush 时得到的 Value 写入 Hot vLog 中，而 GC 后需要重新写回的 Value 都写入 Cold vLog，因为这部分 Value 都是未被新版本覆盖的，可以认为是写入频次较低。

![截屏2021-12-01 下午10.02.45.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/25656933/1638367374251-6e545337-aab4-4029-b4e8-bad500b4da94.png)

# 实验

实验在一台内存大小为 16GB 的服务器上进行，内存一半用于 Block Cache，其它作为系统的 Page Cache 使用。磁盘使用的是一块 480GB 的 SATA III SSD。

LSM-Tree 的配置如下，MemTable 大小为 64MB，第一层大小为 64MB，SSTable 大小为 16MB，vTable 大小为 8MB，然后小、中、大 Value 的阈值分别为 128B 和 8KB。此外 Titan 的配置有几种，NO-GC 表示完全不进行 GC，性能最好，但是空间浪费最严重；BG-GC 表示由后台线程进行 GC；FG-GC 同样由后台线程进行 GC，只不过当空间浪费过大时 FG-GC 会阻塞前台的写入操作。

文中测试使用的 Workload 中，Key 的大小是 24B，Value 的平均大小是 1KB，符合 Pareto 分布；读写热点的分布是 Zipf 分布。测试时先载入 100GB 的数据，然后分别执行 Insert、Update、Read 和 Scan 操作。

结果如图所示，分别展示了其不同操作的吞吐、延迟。可以看出，DiffKV 在所有不同的操作类型中都有较好的性能。而通过最后一张图可以看出，DiffKV 也保持了较小的空间放大。

![截屏2021-12-01 下午10.03.23.png](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/25656933/1638367416620-3688196c-5318-4bd3-a191-fb3683d44b16.png)

此外，论文还进行了在 YCSB 下的实验，具体结果可以阅读论文相关章节。

# 结语：

我们知道，目前的硬件下基本上都是顺序访问的性能要优于随机访问。因此对于一个数据结构来说，追求写入性能可以认为是在追求时间有序性（比如 Append-Only Log）；而追求查询性能（特别是范围查询），就是追求空间有序性（比如 Sorted Group），而这两者在很多时候是矛盾的。最早 Wisckey 中，vLog 就是一个 Append-Only Log，其写入性能最好，但是从空间上来看完全不具有有序性，因此查询性能最差；而最初的 LSM-Tree 中，KV 存储在一起，Value 在每一层在空间上都是有序的，查询性能最好的同时，Compaction 引起的写放大也是最大的。DiffKV 通过将 Value 存储在部分有序的 vTree 中的方式，在时间有序性和空间有序性中取得一个平衡，因此能够同时拥有较好的读写性能。

# 参考文献

1.  Differentiated Key-Value Storage Management for Balanced I/O Performance
1.  PebblesDB: Building Key-Value Stores using Fragmented Log-Structured Merge Trees
1.  Dostoevsky: Better Space-Time Trade-Offs for LSM-Tree Based Key-Value Stores via Adaptive Removal of Superfluous Merging
1.  WiscKey: Separating Keys from Values in SSD-conscious Storage
1.  HashKV: Enabling Efficient Updates in KV Storage via Hashing



Reference:

