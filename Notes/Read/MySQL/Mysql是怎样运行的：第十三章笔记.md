# Mysql是怎样运行的：第十三章笔记

---

## 两种不同的统计数据存储方式

---

InnoDB 提供两种存储统计数据的方式：

* 永久性的统计数据（统计数据存储在磁盘上）。
* 非永久性的统计数据（统计数据存储在内存中）。对于非永久性的统计数据在服务器关闭时就会被清除，待到服务器重启，在适当的情况下统计数据才会被重新收集。

系统变量 `innodb_stats_persistent` 可以控制存储统计数据的方式，在 MySQL 5.6.6 前，该系统变量默认 OFF ,即采用非永久性的统计数据的方式存储统计数据。后续的版本中该系统变量的值默认是 ON ,即采用永久性的统计数据的方式存储统计数据。

InnoDB 默认以表为单位来收集和存储统计数据。我们可以通过 STATS_PERSISTENT 属性指定表存储统计数据的方式：

```mysql
-- STATS_PERSISTENT = 1，表示采用永久性的统计数据的方式
-- STATS_PERSISTENT = 0，表示采用非永久性的统计数据的方式
-- 创建表时未指定 STATS_PERSISTENT 属性，则默认采用系统变量 innodb_stats_persistent 的值作为该属性的值
CREATE TABLE 表名 (...) Engine=InnoDB, STATS_PERSISTENT = (1|0);
ALTER TABLE 表名 Engine=InnoDB, STATS_PERSISTENT = (1|0);
```

<br />

## 基于磁盘的永久性统计数据

---

将表以及该表索引的统计数据存放到磁盘上，实际上就是把这些统计数据存放到系统数据库 mysql 的两个表中：

```mysql
-- 查询语句
mysql> SHOW TABLES FROM mysql LIKE 'innodb%';

-- 查询结果
-- innodb_table_stats 表用来存储关于表的统计数据，每一条记录对应着一个表的统计数据
-- innodb_index_stats 表用来存储关于索引的统计数据，每一条记录对应着一个索引的一个统计项的统计数据
+---------------------------+
| Tables_in_mysql (innodb%) |
+---------------------------+
| innodb_index_stats        |
| innodb_table_stats        |
+---------------------------+
```

<br />

### innodb_table_stats

---

我们来看一下这张表的结构：

|          字段名          |            描述            |
| :----------------------: | :------------------------: |
|      database_name       |          数据库名          |
|        table_name        |            表名            |
|       last_update        |    本条记录最后更新时间    |
|          n_rows          |       表中记录的条数       |
|   clustered_index_size   | 表的聚簇索引占用的页面数量 |
| sum_of_other_index_sizes | 表的其他索引占用的页面数量 |

这张表的主键是 `(database_name,table_name)`，这也就意味着，innodb_table_stats 表中的每一条记录，代表着一个表的统计信息。

**注意：字段 n_rows、clustered_index_size 和 sum_of_other_index_sizes 都是估值。**

<br />

#### n_rows 统计项的收集

---

为何字段 n_rows 的值是估值？因为 InnoDB 统计一个表中有多少行记录的过程如下：

* 按照一定算法选取几个叶子节点页面，计算每个页面中主键值记录数量，然后计算平均一个页面中主键值的记录数量乘以全部叶子节点的数量就算是该表的 n_rows 值。

真实的过程肯定比上述的讲解复杂，但是大意是一致的。

显然，选取叶子节点页面的数量越多，n_rows 值越精确，但统计耗时长。反之，n_rows 值越不精确，但统计耗时短。

可以通过 `innodb_stats_persistent_sample_pages` 系统变量来控制选取叶子节点页面数量，他的默认值是 20 。也可以在创建或修改表的时候通过指定 STATS_SAMPLE_PAGES 属性来控制单个表的统计数据存储方式：

```mysql
CREATE TABLE 表名 (...) Engine=InnoDB, STATS_SAMPLE_PAGES = 具体的采样页面数量;
ALTER TABLE 表名 Engine=InnoDB, STATS_SAMPLE_PAGES = 具体的采样页面数量;
```

如果创建表的语句中，没有指定 STATS_SAMPLE_PAGES 属性，则采用系统变量`innodb_stats_persistent_sample_pages` 的值作为该属性的值。

<br />

#### clustered_index_size 和 sum_of_other_index_sizes 统计项的收集

---

收集链路如下：

* 从数据字典里找到表的各个索引对应的根页面位置。
* 从根页面的 Page Header 里找到叶子节点段和非叶子节点段对应的 Segment Header 。
* 从叶子节点段和非叶子节点段的 Segment Header 中找到这两个段对应的 INODE Entry 结构。
* 从对应的 INODE Entry 结构中找到该段对应所有零散的页面地址以及 FREE 、NOT_FULL、FULL 链表的基节点。
* 直接统计零散的页面有多少个，然后从三个链表的 List Length 字段中读出该段占用的区的大小，每个区占用 64 个页，所以就可以统计出整个段占用的页面。
* 分别计算聚簇索引的叶子结点段和非叶子节点段占用的页面数，它们的和就是 clustered_index_size 的值，按照同样的套路把其余索引占用的页面数都算出来，加起来之后就是 sum_of_other_index_sizes 的值。

这样看来 clustered_index_size 和 sum_of_other_index_sizes 统计项的收集，数据应该是十分精确的，但实际上 clustered_index_size 和 sum_of_other_index_sizes 统计项的值确是估值，为什么呢？

我们知道一个段的数据在非常多时（超过32个页面），会以区为单位来申请空间，**但是以区为单位申请空间中有一些页可能并没有使用**，统计 clustered_index_size 和 sum_of_other_index_sizes 的值时却会吧这些没有使用的页算进去，所以说聚簇索引和其他的索引占用的页面数可能比这两个值要小一些。

**注意：强烈建议结合第九章笔记一同阅读当前段落。**

<br />

#### innodb_index_stats

---