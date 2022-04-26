# 背景

![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/314457/1638712172791-dc26827f-c68d-4ec3-a679-1680e0ac3208.png) 
看到以上的场景，可能不少 DBA 都会感同身受；同时，老板还可能会担心 DBA 的人品，就比如去年才发生了某科技公司删库导致业务瘫痪的事件。当然现在连很多小公司或开发团队也懂得象征性地设置一些数据库账号权限（不少时候可能会为了便捷而混用权限账号），但如果对于已经初具规模的企业来说的话，即使已经严格划分为有 SUPERUSER 权限的 DBA 账号和按需授权的开发账号，老板仍会担心 DBA 有权限访问业务敏感数据、或者授权给不明身份人士、甚至可以通过审计日志看到不同业务部门对数据库的操作，总之“只手遮天”的 DBA 仿佛就是把握了企业数据的命脉。

针对以上问题，PolarDB-X 新增支持三权分立模式，打破了传统数据库运维由 DBA 行使特权的独立控制体系，使得数据库管理员 DBA、安全管理员 DSA 和审计管理员 DAA 三者的权责更加清晰。其中：

-   数据库管理员（DBA， Database Administrator）：只具备 DDL 权限
-   安全管理员（DSA，Department Security Administrator）：只具备管理角色（Role）或用户（User）以及为其他账号授予权限的权限
-   审计管理员（DAA，Data Audit Administrator）：只具备查看审计日志的权限

该功能特性强调的是通过访问层次上的控制体系，来从数据库层面降低安全风险。

# 原理简介

以 Oracle 这一款王者级别的数据库管理系统为例，其从 Oracle 10g 开始推出了 Database Vault [2]这一运维安全体系框架。在这套框架内，其定义了数据库中以下三种核心职责：

-   **账号管理**：包括创建、修改、删除用户账号
-   **安全管理**：设置账户对数据库的访问权限，授权用户对数据库表上可执行的操作
-   **资源管理**：管理数据库系统的资源，但无权访问业务数据，比如进行定期备份、性能监控与调优等操作

可以看出，其核心思想是将原来单一的数据库管理员超级权限进行拆分。在此基础上，PolarDB-X 的三权分立管理模式则结合了分布式数据库的特点，将权限信息保存在独立的 GMS 元数据中心，并且权限职责划分得更加清晰明确，下表展示了三权分立与默认模式的权限区别：

<table>
<tr class="header">
<th>权限</th>
<th>默认模式</th>
<th>三权分立-<strong>DBA</strong></th>
<th>三权分立-<strong>DSA</strong></th>
<th>三权分立-<strong>DAA</strong></th>
</tr>
<tr class="odd">
<td>DML</td>
<td>✔️</td>
<td>❌</td>
<td>❌</td>
<td>❌</td>
</tr>
<tr class="even">
<td>DQL</td>
<td>✔️</td>
<td>❌</td>
<td>❌</td>
<td>❌</td>
</tr>
<tr class="odd">
<td>DAL</td>
<td>✔️</td>
<td>❌</td>
<td>❌</td>
<td>❌</td>
</tr>
<tr class="even">
<td>DDL</td>
<td>✔️</td>
<td>✔️</td>
<td>❌</td>
<td>❌</td>
</tr>
<tr class="odd">
<td>账号角色管理</td>
<td>✔️</td>
<td>❌</td>
<td>✔️</td>
<td>❌</td>
</tr>
<tr class="even">
<td>审计日志</td>
<td>✔️</td>
<td>❌</td>
<td>❌</td>
<td>✔️</td>
</tr>
</table>

其中，具体的权限分类如下所示：

-   DML 包括 INSERT、UPDATE、DELETE语句权限
-   DQL 包括 SELECT语句权限
-   DAL 包括 EXPLAIN、SHOW、sql限流等语句权限
-   DDL 包括 CREATE TABLE / VIEW / INDEX、DROP TABLE / VIEW / INDEX等语句权限
-   账号角色管理包括 CREATE USER / ROLE、GRANT、REVOKE 等语句权限
-   审计日志则包括对 information_schema 下 polardbx_audit_log 系统表的访问权限

# 功能场景展示

**功能开启**
开启三权分立模式均为白屏化操作

1.  在 PolarDB-X 控制台开启三权分立：

![控制台开启](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/314457/1638356782172-9356d007-698a-4e99-8878-51b3c94bccd5.png)

1.  依次创建安全管理员账号和审计管理员账号：

![创建DSA](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/314457/1638336258047-c5e732b3-b757-4893-aab5-b21fcd9b103b.png)

![创建DAA](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/314457/1638336309751-c2b9ac85-1b94-42ac-951f-5121818332d2.png) 
3. 创建成功后可以看到三个管理员账号：
![创建成功](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/314457/1638431124510-6e6bd114-b4b7-481b-9633-f0ad31bfbe84.png)

下面将将结合具体的场景、以上述三个类型管理员的身份来演示不同类型的SQL操作，并展示三权分立支持的核心功能，其中三类管理员的账号名分别如下所示：

-   系统管理员：admin_dba
-   安全管理员：admin_security
-   审计管理员：admin_audit

**场景一 隔离敏感数据**
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/314457/1638714620980-ba61e6a7-0018-4bee-9b76-3de5c075c2a0.png)

```sql
-- 登录admin_dba
mysql> select USER();
+---------------------------+
| USER()                    |
+---------------------------+
| admin_dba@100.120.138.141 |
+---------------------------+
1 row in set (0.00 sec)

mysql> select * from orders;
ERROR 5108 (HY000): [13aaee05c5003000][11.115.153.234:3035][tmall]ERR-CODE: [TDDL-5108][ERR_CHECK_PRIVILEGE_FAILED_ON_TABLE] User admin_dba@'100.120.138.141' does not have 'SELECT' privilege on table 'ORDERS'. Database is TMALL.

mysql> insert into orders (`o_id`,`o_c_id`,`o_s_id`) values (162103411, 1, 123);
ERROR 5108 (HY000): [13aaee0c37803000][11.115.153.234:3035][tmall]ERR-CODE: [TDDL-5108][ERR_CHECK_PRIVILEGE_FAILED_ON_TABLE] User admin_dba@'100.120.138.141' does not have 'INSERT' privilege on table 'ORDERS'. Database is TMALL.
```

**场景二 隔离权限授予**
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/314457/1638714634777-117b1ec6-1335-45e8-8fa7-50f29b24b04b.png)

```sql
-- 登录admin_dba
mysql> select USER();
+---------------------------+
| USER()                    |
+---------------------------+
| admin_dba@100.120.138.141 |
+---------------------------+
1 row in set (0.00 sec)

mysql> CREATE USER 'user1'@'%' IDENTIFIED BY 'polarX123456';
ERROR 5110 (HY000): [13aaee115fc03000][11.115.153.234:3035][tmall]ERR-CODE: [TDDL-5110][ERR_CHECK_PRIVILEGE_FAILED] User admin_dba@'100.120.138.141' does not have 'CREATE ACCOUNT' privilege. Database is TMALL.

-- 登录admin_security
mysql> select USER();
+--------------------------------+
| USER()                         |
+--------------------------------+
| admin_security@100.120.137.141 |
+--------------------------------+
1 row in set (0.00 sec)

mysql> CREATE USER 'tmall_dev'@'192.168.0.%' IDENTIFIED BY 'polarX123456';
Query OK, 0 rows affected (0.07 sec)

mysql> grant select,insert,update on tmall.* to 'tmall_dev'@'192.168.0.%';
Query OK, 0 rows affected (0.03 sec)

mysql> show grants for 'tmall_dev'@'192.168.0.%';
+----------------------------------------------------------------------+
| GRANTS FOR 'TMALL_DEV'@'192.168.0.%'                                 |
+----------------------------------------------------------------------+
| GRANT USAGE ON *.* TO 'tmall_dev'@'192.168.0.%'                      |
| GRANT SELECT, INSERT, UPDATE ON tmall.* TO 'tmall_dev'@'192.168.0.%' |
+----------------------------------------------------------------------+
2 rows in set (0.00 sec)

```

**场景三 隔离sql日志**
![undefined](https://intranetproxy.alipay.com/skylark/lark/0/2021/png/314457/1638715871281-255fca01-ea09-49b9-992c-93a763b25888.png)

```sql
-- 登录admin_audit
mysql> select USER();
+-----------------------------+
| USER()                      |
+-----------------------------+
| admin_audit@100.120.138.141 |
+-----------------------------+
1 row in set (0.00 sec)

mysql> use information_schema;
Database changed
mysql> select USER_NAME,HOST,PORT,AUDIT_INFO,ACTION,TRACE_ID from polardbx_audit_log where SCHEMA = 'tmall';
+----------------+-----------------+------+--------------------------------------------------------------------+-------------+------------------+
| USER_NAME      | HOST            | PORT | AUDIT_INFO                                                         | ACTION      | TRACE_ID         |
+----------------+-----------------+------+--------------------------------------------------------------------+-------------+------------------+
| admin_security | 100.120.137.141 | 9635 | CREATE USER 'tmall_dev'@'192.168.0.%' IDENTIFIED BY 'polarX123456' | CREATE_USER | 13aaef15fd004000 |
| admin_security | 100.120.137.141 | 9635 | grant select,insert,update on tmall.* to 'tmall_dev'@'192.168.0.%' | GRANT       | 13aaef5d78804000 |
+----------------+-----------------+------+--------------------------------------------------------------------+-------------+------------------+
2 rows in set (0.01 sec)
```

关于详细使用方法与实践可参考[官方文档](https://help.aliyun.com/document_detail/213823.html)。

# 总结

回过头来看，三权分立是不是能彻底解决在开篇背景中提到的删库跑路的问题呢？事实上，看完这篇文章，你会知道并不能；如果想知道如何避免删库等操作带来的风险，可以参考系列文章《[PolarDB-X 如何拯救误删数据的你](https://zhuanlan.zhihu.com/p/367137740)》。

那么三权分立究竟带来了什么呢？三权分立本质就是权力的分散，而这并不意味着会导致效率的低下；相反实行分责制度，避免一些误操作的风险，还可以有效降低敏感数据泄露的风险。

笔者相信，在数据保护越来越严格、企业操作规范越来越健全的如今，三权分立模式将为需要管理大规模数据库的企业降低运维风险、提供敏感数据访问保护，从而企业不仅能够同时减少来自内、外部的威胁，还能满足相应的合规需求。

---

# 参考

1.  https://docs.oracle.com/en/database/oracle/oracle-database/21/dvadm/oracle-database-vault-schemas-roles-and-accounts.html#GUID-4E1707B4-F9BB-4131-9D3E-90BB273F2832
1.  https://dev.mysql.com/doc/refman/8.0/en/privileges-provided.html



Reference:

