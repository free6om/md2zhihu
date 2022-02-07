#### 说明

PolarDB-X 逻辑上由计算节点（CN）、存储节点（DN）、元数据节点（GMS）和全局日志节点（CDC）组成，代码仓库包括 GalaxySQL、GalaxyGlue、GalaxyEngine、GalaxyCDC和 GalaxyKube 等。
PolarDB-X 采用总体 3 个月为周期、每个仓库独立版本号的策略进行迭代。

#### 版本更新

本次更新是继 2021 年 10 月 20 号云栖大会正式开源后的第一次版本更新，更新内容包括新增**读写分离**、**集群扩缩容**等特性，兼容 maxwell 和 debezium 增量日志订阅，以及新增其他众多新特性和修复若干问题。
本次更新详细内容如下：

<table>
<tr class="header">
<th>仓库名称</th>
<th>版本号</th>
<th>更新内容</th>
</tr>
<tr class="odd">
<td>GalaxySQL</td>
<td>5.4.12</td>
<td>见 <a href="https://github.com/ApsaraDB/galaxysql/releases/tag/galaxysql-5.4.12">Release Note</a></td>
</tr>
<tr class="even">
<td>GalaxyEngine</td>
<td>20220127</td>
<td>见 <a href="https://github.com/ApsaraDB/galaxyengine/commits/main">commit log</a></td>
</tr>
<tr class="odd">
<td>GalaxyCDC</td>
<td>5.4.12</td>
<td>见 <a href="https://github.com/ApsaraDB/galaxycdc/releases/tag/galaxycdc-5.4.12">Release Note</a></td>
</tr>
<tr class="even">
<td>GalaxyKube</td>
<td>1.1.0</td>
<td>见 <a href="https://github.com/ApsaraDB/galaxykube/tree/main/CHANGELOG#2022-01-27">ChangeLog</a></td>
</tr>
</table>



Reference:

