# Mysql是怎样运行的：第十六章笔记

---

## 执行计划输出中各列详解（下）

---

### Extra

---

Extra 即**额外信息**，额外信息可以帮助我们更准确的理解 MySQL 如何执行给定的查询语句。额外信息不止一个，**以下我仅列举出常见的或比较重要的额外信息**。

<br />

#### No tables used

---

当查询语句没有 FROM 子句时，Extra 字段的值含有 No tables used 提示。

<br />

#### Impossible WHERE

---

当查询语句的 WHERE 子句永为 FALSE 时，Extra 字段的值含有 Impossible WHERE 提示。

<br />

#### No matching min/max row

---

当查询语句的查询列表处有 MIN / MAX 聚集函数，但是并没有符合 WHERE 子句的搜索条件的记录时，Extra 字段的值含有 No matching min/max row 提示。

<br />

#### Using index

---

当查询可以使用索引覆盖的情况下，Extra 字段的值含有 Using index 提示。

<br />

#### Using index condition

---

当查询语句的执行过程中使用到了**索引条件下推（Index Condition Pushdown）**时，Extra 字段的值含有 Using index condition 提示。

何谓索引条件下推，以下举例说明。

```mysql
-- 查询语句搜索条件出现了索引列，但是对于条件" key1 LIKE '%a' "而言，无法使用到索引 idx_key1
SELECT * FROM s1 WHERE key1 > 'z' AND key1 LIKE '%a';
```

按正常流程，应先根据搜索条件`key1 > 'z'`，从二级索引 idx_key1 中获取到对应的二级索引记录后回表找到完整的用户记录，再检测对应的用户记录是否符合搜索条件`key1 LIKE '%a'`，最后将符合条件用户记录加入到结果集中。

但是对于索引条件下推，在根据搜索条件`key1 > 'z'`，从二级索引 idx_key1 中获取到对应的二级索引记录后并不立即回表，而是利用搜索条件`key1 LIKE '%a'`过滤二级索引记录后再回表，这便是**索引条件下推**。

<br />

#### Using where

---

两种情况下，Extra 字段的值含有 Using where 提示。

1. 当使用全表扫描执行对某个表的查询，并且查询语句的 WHERE 子句中有针对该表的搜索条件时。
2. 当使用索引访问执行对某个表的查询，并且查询语句的 WHERE 子句中，除了该索引包含的列，还有其他的搜索条件时。

<br />

#### Using join buffer (Block Nested Loop)

---

在连接查询执行过程中，若被驱动表不能有效的利用索引加快访问速度，MySQL 一般会为其分配一块名叫`join buffer`的内存块来加快查询速度（即基于块的嵌套循环连接） ，这时，Extra 字段的值含有 Using join buffer (Block Nested Loop) 提示。

例如：

```mysql
--  EXPLAIN 语句
--  连接查询被驱动表 s2 无法有效利用索引，只好使用 join buffer 来减少对 s2 表的访问次数，这便是执行计划中，Extra 字段值 Using join buffer (Block Nested Loop) 的由来。
--  连接查询在访问 s2 表时（进行全表扫描），s1.common_field的值已确定，所以实际上查询 s2 表的条件是：s2.common_field = 常数，这便是执行计划中，Extra 字段值 Using where 的由来。
EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.common_field = s2.common_field;

-- 执行计划
+---+-----------+-----+----------+----+-------------+----+-------+----+----+--------+---------------------------------------------------+
|id |select_type|table|partitions|type|possible_keys|key |key_len|ref |rows|filtered|Extra                                              |
+---+-----------+-----+----------+----+-------------+----+-------+----+----+--------+---------------------------------------------------+
| 1 | SIMPLE    | s1  | NULL     |ALL | NULL        |NULL| NULL  |NULL|9688|100.00  |NULL                                               |
| 1 | SIMPLE    | s2  | NULL     |ALL | NULL        |NULL| NULL  |NULL|9954|10.00   |Using where; Using join buffer (Block Nested Loop) |
+---+-----------+-----+----------+----+-------------+----+-------+----+----+--------+---------------------------------------------------+

```

<br />

#### Not exists

---

当查询语句使用左（外）连接时，若搜索条件中要求被驱动表的某个列为 NULL，但这个列不允许存储 NULL 值时，Extra 字段的值含有 Not exists 提示。

以上述概念为例，当驱动表的一条记录，在被驱动表中找不到一条与搜索条件匹配的记录时（即找不到一条被驱动表对应 NOT NULL 的列的值为 NULL 的记录），该条驱动表的记录加入到最终的结果集中。

这句话等价于，当驱动表的一条记录，在被驱动表中找到一条与搜索条件匹配的记录时（即找到一条被驱动表对应 NOT NULL 的列的值为 NULL 的记录），该条驱动表的记录不会加入到最终的结果集中。

也就是说， MySQL 在执行 Not exists 这类查询时，一旦在被驱动表中找到一条与搜索条件匹配的记录时，就会停止继续到被驱动表中搜索其他可能的匹配项，因为此时驱动表对应的记录，不会被加入到最终的结果集中。而且明显的，这样做可以提高查询效率。

**注意：因为左（外）连接可以转换为右（外）连接，所以右（外）连接同理。**

<br />

#### Using intersect(...) 、Using union(...) 和 Using sort_union(...)

---

当准备使用 Intersect 索引合并的方式执行查询，Extra 字段的值含有 Using intersect(...) 提示。

当准备使用 Union 索引合并的方式执行查询，Extra 字段的值含有 Using union(...) 提示。

当准备使用 Sort-Union 索引合并的方式执行查询，Extra 字段的值含有 Using sort_union(...) 提示。

上述提示中，`...`表示需要进行索引合并的索引名称。

<br />

#### Zero limit

---

当查询语句的 LIMIT 子句参数为 0 时（即查询语句不打算从表中读出任何记录时）， Extra 字段的值含有 Zero limit 提示。

<br />

#### Using filesort

---

对结果集中的记录进行排序，是可以使用到索引的。但当排序操作无法使用到索引时，就只可以在内存中（记录较少时）或磁盘中（记录较多时）进行排序，这种在内存或磁盘上进行排序的方式统称为**文件排序（英文名：filesort）**。

当查询语句需要使用到文件排序的方式执行查询时，Extra 字段的值就会含有 Using filesort 提示。

明显的，如果查询中需要使用 filesort 的方式进行排序的记录非常多，那么这个过程是很耗费性能的。最好将使用 filesort 的执行方式改为使用索引进行排序。

<br />

#### Using temporary

---

当查询可能使得 MySQL 借助临时表来完成一些功能（例如去重、排序，或执行许多包含`DISTINCT`、`GROUP BY`、`UNION`关键字，但不可有效利用索引来完成查询的子句），Extra 字段的值就会含有 Using temporary 提示。

**MySQL 会在包含 GROUP BY 子句查询中默认添加上 ORDER BY 子句（GROUP BY 哪个字段就对哪个字段排序）。**

也就是说，当查询使得 MySQL 借助临时表来完成一些功能，并且查询语句中包含了 GROUP BY 子句，可能会因为 GROUP BY 子句的默认排序行为，导致 Extra 字段的值含有 Using filesort 提示（非索引列排序），这时候查询就会增加对记录进行文件排序的成本。

我们可以在查询语句中显式的写上 ORDER BY NULL 语句，表示不想为包含 GROUP BY 子句的查询进行排序。这样，就意味着执行查询时可以省去对记录进行文件排序的成本了。

<br />

#### Start temporary, End temporary

---



























