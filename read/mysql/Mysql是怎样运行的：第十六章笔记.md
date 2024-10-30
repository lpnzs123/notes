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

当查询语句没有 FROM 子句时，Extra 字段的值为 No tables used。

<br />

#### Impossible WHERE

---

当查询语句的 WHERE 子句永为 FALSE 时，Extra 字段的值为 Impossible WHERE。

<br />

#### No matching min/max row

---

当查询语句的查询列表处有 MIN / MAX 聚集函数，但是并没有符合 WHERE 子句的搜索条件的记录时，Extra 字段的值为 No matching min/max row。

<br />

#### Using index

---

当查询可以使用索引覆盖的情况下，Extra 字段的值为 Using index。

<br />

#### Using index condition

---

当查询语句的执行过程中使用到了**索引条件下推（Index Condition Pushdown）**时，Extra 字段的值为 Using index condition。

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

两种情况下，Extra 字段的值为 Using where。

1. 当使用全表扫描执行对某个表的查询，并且查询语句的 WHERE 子句中有针对该表的搜索条件时。
2. 当使用索引访问执行对某个表的查询，并且查询语句的 WHERE 子句中，除了该索引包含的列，还有其他的搜索条件时。

<br />

#### Using join buffer (Block Nested Loop)

---

在连接查询执行过程中，若被驱动表不能有效的利用索引加快访问速度，MySQL 一般会为其分配一块名叫`join buffer`的内存块来加快查询速度（即基于块的嵌套循环连接） ，这时，Extra 字段的值为 Using join buffer (Block Nested Loop)。

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































