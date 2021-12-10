# 背景

在现今数据即资产的大数据时代，各行各业对数据存储都有着自己的监管标准，而这些标准很多都一致地指出了“用户个人数据必须加密存储”这一条目，包括耳熟能详的 GDPR<sup>[1]</sup>，还有 CCPA<sup>[2]</sup>、PCI DSS<sup>[3]</sup>等条例；以及最近中国实施的《个人信息保护法》中也指出，应对个人信息采取相应的加密、去标识化等安全技术措施。因此，越来越多企业用户希望自己上云的数据能够加密存储，并满足行业监管与安全合规需求。

为了支持数据加密存储这一功能，许多数据库管理系统在数据库层面就实现了 TDE（Transparent Data Encryption），即**透明数据加密**。其在不依赖文件系统文件级加密的情况下，不仅可以保证硬盘上存储的数据为密文，还可以控制加密粒度（库、表、列）来降低加密后性能上的影响。

PolarDB-X 作为一款云原生分布式数据库，支持云端的数据加密自然是一项赢得客户信任的关键能力，本文将从原理与应用这两个方面来介绍 PolarDB-X 的 TDE 这一企业级特性。

# 原理简介

透明数据加密的本质就是对数据库的物理文件进行密文存储，黑客无法从物理文件直接读取到敏感数据，而“透明”则体现在数据的加解密是由数据库内核完成，用户没有感知，也无需做任何业务代码上的修改。PolarDB-X 的 TDE 特性是在 InnoDB 存储引擎静态数据加密<sup>[4]</sup>能力的基础上，提供了分布式场景下与 MySQL 兼容的语法支持，并结合了阿里云密钥管理服务KMS来实现的。

**什么是静态数据加密**

讨论到数据加密，其场景可分为三类：

-   存储加密，即对存储上的物理文件进行加密；
-   传输加密，即对网络信道传输的数据进行加密；
-   计算加密，即对内存、CPU缓存或寄存器中的数据进行加密。

而静态数据加密正属于存储加密的一种，也就是会对数据库存储数据的物理文件进行加密。

**加密对象**

既然是加密物理文件，部分对 InnoDB 有一定了解同学可能会认为，是不是直接将".ibd"文件加密就满足需求了呢？但事实上，InnoDB 的静态数据加密（Data-at-rest encryption）支持以下文件对象：

-   file-per-table 表空间
-   file-per-table 表空间
-   genera l表空间
-   doublewrite file 表空间
-   system 表空间
-   redo log
-   undo log
-   binlog与relay log
-   audit log

可以发现，除了表空间的数据支持被加密，各种包含数据的日志文件也支持被加密；下文将主要以 file-per-table 表空间为例来介绍加解密流程。

**何时加密/解密**

InnoDB数据加密只会发生在数据落盘的时刻，也就是说数据页 (page)在内存的 buffer pool 中都是明文状态，只有当刷脏时，才会由后台刷脏线程使用表空间对应的密钥对数据进行加密并写入磁盘。显然由于buffer pool中的数据都是解密后的状态，存储引擎读取物理文件时，是先解密再将数据放入内存。

所以实际上在返回客户端数据时，网络传输的也都是解密后的数据，那么此时的安全性则需要SSL/TLS特性来保障，PolarDB-X 对此也提供了支持。

**如何加密**

InnoDB 使用了标准 AES 加密算法，其中表空间的密钥是使用 AES-256-ECB 算法加密，数据页则使用 AES-256-CBC 算法加密。AES 为块加密算法，需要对明文按照128位进行分块，每一个块加密后，再把一块块的密文合起来形成最终密文结果；并且作为块加密算法，密文的长度与明文数据在对齐 (padding)后一致，因此加密数据的存储大小并不会膨胀。

同时由于 PolarDB-X 整合了阿里云密钥管理服务 (KMS)的能力，除了 AES 加密算法以外，还能提供国密算法，如SM4、SM3，进而满足更多的客户场景。

此外，整体加密的安全性不仅取决于加密算法的安全性，还包括加解密时使用密钥的安全性。事实上，MySQL TDE 方案就是 InnoDB 静态数据加密 + 可靠的密钥管理方案。其在密钥管理上采用了双层密钥架构，即包括主密钥与表空间密钥。其中主密钥由中心化密钥管理服务存储，负责加密生成表空间密钥，然后将表空间密钥保存在表空间的头部(0号页面)；而表空间密钥则用来真正加解密表空间文件<sup>[5]</sup>。由于表空间密钥本身需要主密钥来解密，MySQL 正是通过这种密钥管理方案来做到密钥与加密数据的分离。

<div style="text-align: center;">
  <img src="https://intranetproxy.alipay.com/skylark/lark/0/2021/png/314457/1636338195379-a6f45211-4caa-4c6f-82a0-023f8c0c468a.png"  style="zoom: 60%;" />
</div>
<p style="text-align: center;">双层密钥架构</p>

下面两图展示了 MySQL TDE方案与 PolarDB-X TDE方案的架构。

<div style="text-align: center;">
  <img src="https://intranetproxy.alipay.com/skylark/lark/0/2021/png/314457/1636338149961-dffc80c1-259d-4a34-b720-643c2a81d76c.png"  style="zoom: 55%;" />
</div>
<p style="text-align: center;">MySQL TDE方案</p>

<div style="text-align: center;">
  <img src="https://intranetproxy.alipay.com/skylark/lark/0/2021/png/314457/1636338231995-af810287-7c4b-4167-9339-fce782ac6c6f.png"  style="zoom: 60%;" />
</div>
<p style="text-align: center;">PolarDB-X TDE方案（CN 为计算节点，DN 为存储节点）</p>

# 使用简介

PolarDB-X TDE 的加密粒度为表级别，且支持分区表、支持scale out，即在透明分布式的基础上实现透明加密的特性；目前对MySQL语法兼容情况如下：

-   [X] CREATE TABLE [table_name] ([col definition]) ENCRYPTION='Y';
-   [X] ALTER TABLE [table_name] ENCRYPTION='Y';
-   [X] ALTER TABLE [table_name] ENCRYPTION='N';
-   [ ] ALTER INSTANCE ROTATE INNODB MASTER KEY

**开启TDE**

PolarDB-X 使用 TDE 特性非常方便快捷，相比较起 MySQL，后者还需要安装 keyring 插件、配置密钥文件等操作。

1.  在 PolarDB-X 控制台开启 TDE（配置与管理-安全管理-TDE）：

![控制台TDE开启](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/314457/1636522654886-02dc4ae4-683e-49ec-8c64-d56cdbd027cd.png) 
2. 根据提示选择自动生成密钥，或者上传自定义密钥：
![密钥设置](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/314457/1636339086718-7fc18577-39cb-4a3c-9707-4b5cf9ec531f.png) 
3. 此时实例状态处于 TDE 修改中，等待修改完毕，即可体验：
![实例状态修改中](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/314457/1636339155206-0931c51a-5d5a-4e98-a794-707542ff660e.png) 
4. 同 MySQL 语法一样，创建加密表并使用（支持[分区表语法](https://help.aliyun.com/document_detail/316591.html)）：

```sql
mysql> CREATE TABLE t_tde_test (id int, value varchar(20)) ENCRYPTION='Y';

mysql> INSERT INTO t_tde_test VALUES (1, 'PolarDB-X');

mysql> SELECT * FROM t_tde_test;
+----+-----------+
| id | value     |
+----+-----------+
|  1 | PolarDB-X |
+----+-----------+
```

1.  支持将已经之前创建好且有数据的表变更为加密表：

```sql
mysql> ALTER TABLE t_orders ENCRYPTION='Y';
```

1.  也支持将加密表解密：

```sql
mysql> ALTER TABLE t_tde_test ENCRYPTION='N';
```

由于透明数据加密的特性，使用起来与普通未加密的表没有任何区别。

**性能影响**

一些用户可能会担心开启 TDE 后对性能的影响，下图展示了PolarDB-X 实例在开启 TDE 前后，以 TPC-C 100仓作为基准测试的性能对比结果（横轴为并发线程数，纵轴为 tmpC）：

![TPCC](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/314457/1637116684746-8c5107ed-18dc-4047-86cd-d7b49175bc28.png) 
可以看到，开启 TDE 后，大部分情况下性能与关闭状态没有差异，仅个别情况会有小幅度的性能回退（约5%以内）；从上述原理分析来看，透明数据加密主要是在文件 I/O 时对 CPU 有额外的开销，且整体 I/O 的大小并不会膨胀，性能下降是比较有限的。因此，TDE 完全具备在生产环境中应用的条件。

# 总结

PolarDB-X 作为一款云原生分布式数据库产品，具备了透明数据加密的企业级特性，不仅在语法上兼容 MySQL 生态，用户的应用代码也无需做任何改造，并且在加密算法上更是支持国产密码算法。因此在如今行业监管越来越规范严格的趋势下，PolarDB-X 的企业级特性 TDE 将能满足更多客户的需求。我们也欢迎大家体验 PolarDB-X 更多的企业级特性！

---

# 参考

1.  欧盟《一般数据保护条例》第32条，https://gdpr-info.eu/art-32-gdpr/
1.  《加州消费者隐私保护法》1798.150.a，https://www.spirion.com/wp-content/uploads/2020/07/Spirion_CCPA_v3.pdf
1.  《支付卡行业数据安全标准》Requirement3，https://www.pcisecuritystandards.org/documents/PCI_DSS-QRG-v3_2_1.pdf
1.  https://dev.mysql.com/doc/refman/8.0/en/innodb-data-encryption.html#innodb-data-encryption-about
1.  https://www.percona.com/resources/technical-presentations/how-transparent-data-encryption-built-mysql-and-percona-server



Reference:

