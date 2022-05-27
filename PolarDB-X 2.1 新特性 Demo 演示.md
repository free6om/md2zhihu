PolarDB-X 于 5 月 25 日开源 2.1 版本发布会中详细介绍了 X-Paxos、透明分布式、MySQL 生态兼容等特性，以下是发布会中关于 Flashback Query、GSI、Global Binlog、Replica 的 4 个 Demo 演示视频。

#### 通过 Flashback Query 获取历史版本数据

本 Demo 以转账测试场景为例，展示了通过 SQL 查询数据历史版本、与历史数据进行 JOIN 操作以及通过 SQL 恢复误删除的数据等内容。

[embed: demo1_flashback_query.mp4](https://intranetproxy.alipay.com/skylark/lark/0/2022/mp4/46860/1653630567071-396f981e-65bf-413c-9029-f98d24eddca3.mp4)

#### 使用 GSI 优化查询性能

本 Demo 以 Sysbench 压测场景为例，展示了通过 SQL Advisor（索引推荐） + GSI （全局二级索引）来优化查询性能等内容。

[embed: demo2_使用GSI优化查询性能.mp4](https://intranetproxy.alipay.com/skylark/lark/0/2022/mp4/46860/1653630913842-0a8abb90-a103-4522-860f-a5dc1ca56c91.mp4)

#### 用 Global Binlog + Flink CDC 模拟双十一交易大屏

本 Demo 用 Global Binlog 与 Flink CDC 两项基础能力，搭建了一个实时计算链路，来模拟双十一的 GMV 大屏。

[embed: demo3_global-binlog+flink-cdc.mp4](https://intranetproxy.alipay.com/skylark/lark/0/2022/mp4/46860/1653631075293-b2892ea6-20ba-4e20-be26-7752af6e3bc1.mp4)

#### 用 Replica 搭建 MySQL + PolarDB-X 的主备复制链路

PolarDB-X 支持与 MySQL 体验一致的 Replication 能力，成为 Replica。本 Demo 通过 Replica 能力搭建了一条 MySQL 为主、PolarDB-X 为备的主从复制链路。

[embed: demo4_replication.mp4](https://intranetproxy.alipay.com/skylark/lark/0/2022/mp4/46860/1653631278158-d0c83d72-cb39-4c22-aa9c-58180d184808.mp4)



Reference:

