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

### innodb_index_stats

---

innodb_index_stats 表结构如下：

|      字段名      |              描述              |
| :--------------: | :----------------------------: |
|  database_name   |            数据库名            |
|    table_name    |              表名              |
|    index_name    |             索引名             |
|   last_update    |      本条记录最后更新时间      |
|    stat_name     |          统计项的名称          |
|    stat_value    |        对应的统计项的值        |
|   sample_size    | 为生成统计数据而采样的页面数量 |
| stat_description |       对应的统计项的描述       |

innodb_index_stats 表的主键是`(database_name,table_name,index_name,stat_name)`，**这也就意味着 innodb_index_stats 表的每条记录代表着一个索引的一个统计项。**

接下来我们直接来看看一个索引都有哪些统计项吧。

* n_leaf_pages：表示该索引的叶子节点占用多少页面。
* size：表示该索引共占用多少页面。
* n_diff_pfx**NN**：表示对应的索引列不重复的值有多少。

在以上统计项中，最不好理解的应该就是最后一个了，**NN** 是什么意思？

它其实可以被替换成 01、02、03这样的数字，接下来我们详解一下统计项 n_diff_pfx**NN** ，以 idx_key_part 联合索引为例：（idx_key_part 联合索引构成为` KEY idx_key_part(key_part1, key_part2, key_part3)`）

* n_diff_pfx01 表示统计 key_part1这一个列不重复的值有多少。
* n_diff_pfx02 表示统计 key_part1、key_part2 这两个列组合起来不重复的值有多少。
* n_diff_pfx03 表示的是统计 key_part1、key_part2、key_part3 这三个列组合起来不重复的值有多少。
* n_diff_pfx04 表示的是统计 key_part1、key_part2、key_part3、id 这四个列组合起来不重复的值有多少。

**注意，不唯一的二级索引（即索引列值不唯一）会多一次统计索引列和主键值组合起来不重复的值有多少的情况。**例如：idx_key1有 n_diff_pfx01 、n_diff_pfx02 两个统计项。而 idx_key2 却只有 n_diff_pfx01 一个统计项。

在计算某些索引列中包含多少不重复值时，需要对一些叶子节点页面进行采样。**特别的，对于联合索引而言，采样的页面数量是：innodb_stats_persistent_sample_pages × 索引列的个数。当采样的页面数量大于该索引的叶子节点数量时，就会采用全表扫描来统计该索引不重复值数量。**

上述中采样的页面数量，也就是 size 统计项的值。

<br />

### 定期更新统计数据

---

有两种更新统计数据的方式：

* 开启系统变量`innodb_stats_auto_recalc`。它的默认值时 ON ，也就是默认开启自动重新计算统计数据的功能。每一个表都维护者一个代表对该表进行增删改的记录条数的变量，如果发生变动的记录数量超过的了表大小的 10% ，并且系统变量`innodb_stats_auto_recalc`的值时 ON ，那么服务器就会重新进行一次统计数据的计算，并更新 innodb_table_stats 和 innodb_index_stats 表。这个过程是异步的，也就是说即使表中变动的记录数超过了 10% ，也不会立即自动重新计算统计数据，而是延迟几秒。

  InnoDB 默认以表为单位来收集和存储统计数据也就意味着，我们可以对单个表设置是否自动重新计算统计数，通过 STATS_AUTO_RECALC 属性指明即可。例如：

  ```mysql
  -- STATS_AUTO_RECALC = 1 表明该表会自动重新计算统计数，反之不会
  -- 若未指定 STATS_AUTO_RECALC 属性的值，那么默认采用系统变量 innodb_stats_auto_recalc 的值作为该属性的值
  CREATE TABLE 表名 (...) Engine=InnoDB, STATS_AUTO_RECALC = (1|0);
  
  ALTER TABLE 表名 Engine=InnoDB, STATS_AUTO_RECALC = (1|0);
  ```

* 手动调用`ANALYZE TABLE`语句来更新统计信息。若 系统变量`innodb_stats_auto_recalc`的值为 OFF，则我们可以通过如下方式更新某个表的统计数据，例如：

  ```mysql
  -- 更新语句
  ANALYZE TABLE single_table;
  
  -- 更新结果
  +------------------------+---------+----------+----------+
  | Table                  | Op      | Msg_type | Msg_text |
  +------------------------+---------+----------+----------+
  | xiaohaizi.single_table | analyze | status   | OK       |
  +------------------------+---------+----------+----------+
  ```

  需要注意这个过程是同步的，即 ANALYZE TABLE 语句会立即重新计算统计数据，不要随意的使用 ANALYZE TABLE 语句，因为当表中索引多或者采样的页面特别多的时候这个过程会特别慢。

<br />

### 手动更新 innodb_table_stats 和 innodb_index_stats 表

---

说白了 innodb_table_stats 和 innodb_index_stats 表相当于俩普通的表，那么对他们增删改查也是可以的，**也就是说，我们可以手动更新某个表或者索引的统计数据。**例如，我们更新一下 single_table 表关于行数的统计数据，分两步：

* 更新 innodb_table_stats 表。

  ```mysql
  UPDATE innodb_table_stats 
      SET n_rows = 1
      WHERE table_name = 'single_table';
  ```

* 让 MySQL 查询优化器重新加载我们更改过的数据。使用如下语句

  ```mysql
  FLUSH TABLE single_table;
  ```

之后我们使用`SHOW TABLE STATUS`语句查看表的统计数据时就看到 Rows 行变为了 1 。

<br />

## 基于内存的非永久性统计数据

---

由于非永久性的统计数据经常更新，导致 MySQL 查询优化器计算查询成本的时候依赖于经常变化的统计数据，也就是说会生成经常变化的执行计划。

基于内存的非永久性统计数据因为 MySQL  不怎么使用而不过多的介绍，只要记住非永久性的统计数据采样的页面数量是由`innodb_stats_transient_sample_pages`这个系统变量控制的就行了，它的默认值是 8 。

<br />

## innodb_stats_method 的使用

---











