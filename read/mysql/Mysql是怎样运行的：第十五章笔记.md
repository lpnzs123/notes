# Mysql是怎样运行的：第十五章笔记

---

## 前置准备

---

一条查询语句在经过 MySQL 查询优化器的各种基于成本和规则的优化后，会生成一个所谓的**执行计划**，执行计划展示了接下来具体执行查询的方式。通过 **EXPLAIN** 关键字，我们可以查看某个查询语句的具体执行计划，从而可以针对性的提升我们查询语句的性能。例如：

```mysql
-- 使用 EXPLAIN 关键字查看某个查询语句的具体执行计划
-- 除了 SELECT 语句可以使用 EXPLAIN 关键字，DELETE、INSERT、REPLACE 以及 UPDATE 语句也可以使用 EXPLAIN 关键字来查看这些语句的执行计划
EXPLAIN SELECT 1;
```

该 SQL 会输出一个表（即一个 EXPLAIN 语句的结果集），其中的各个列释义大致如下：

|     列名      |                             描述                             |
| :-----------: | :----------------------------------------------------------: |
|      id       | 在一个大的查询语句中，每个 SELECT 关键字都对应一个唯一的 id  |
|  select_type  |                 SELECT 关键字对应查询的类型                  |
|     table     |                             表名                             |
|  partitions   |                        匹配的分区信息                        |
|     type      |                      针对单表的访问方法                      |
| possible_keys |                        可能用到的索引                        |
|      key      |                        实际使用的索引                        |
|    key_len    |                      实际使用的索引长度                      |
|      ref      |    当使用索引列等值查询时，与索引列进行等值匹配的对象信息    |
|     rows      |                    预估需要读取的记录条数                    |
|   filtered    | 针对预估需要读取的记录，经过搜索条件过滤后剩余记录条数的百分比 |
|     Extra     |                        一些额外的信息                        |

现在存在表 single_table ，结构如下：

```mysql
CREATE TABLE single_table (
    id INT NOT NULL AUTO_INCREMENT,
    key1 VARCHAR(100),
    key2 INT,
    key3 VARCHAR(100),
    key_part1 VARCHAR(100),
    key_part2 VARCHAR(100),
    key_part3 VARCHAR(100),
    common_field VARCHAR(100),
    PRIMARY KEY (id),
    KEY idx_key1 (key1),
    UNIQUE KEY idx_key2 (key2),
    KEY idx_key3 (key3),
    KEY idx_key_part(key_part1, key_part2, key_part3)
) Engine=InnoDB CHARSET=utf8;
```

表 s1 和表 s2 和表 single_table 的结构相同，它们各自有 10000 条记录，除 id 列外其余的列都插入随机值。

<br />

## 执行计划输出中各列详解

---

### table

---

无论多复杂的查询语句，里面包含了多少个表，最后都是要对每个表进行单表访问的。MySQL 规定：**EXPLAIN 语句输出的每条记录都对应着某个单表的访问方法，该条记录的table列代表着该表的表名**。

<br />

### id

---

什么情况下在一条查询语句中会出现多个 SELECT 关键字？

有两种情况：

* 查询中包含子查询的情况。
* 查询中包含 UNION 语句的情况。

**查询语句中每出现一个 SELECT 关键字，MySQL 就会为它分配一个唯一的 id 值**。而这个 id 值，就是 EXPLAIN 语句结果集的第一个列。

**对于连接查询**，一个 SELECT 关键字，它的 FROM 子句中可以有多个表，每个表都会对应 EXPLAIN 语句结果集中的一条记录。**但是对于这些记录，其 id 列的值都是相同的。并且，按记录出现在结果集中的顺序，出现在前面的表为驱动表，后面的表为被驱动表。**

**对于包含子查询的查询**，每个 SELECT 关键字都会对应一个唯一的 id 值。

但是注意，**查询优化器可能会对涉及子查询的查询语句进行重写，从而将其转换为连接查询**。若我们想知道查询优化器是否对某个包含子查询的语句进行了重写，我们应该怎么做呢？

直接查看执行计划就可以了，结合我们上述所说，若执行计划结果集中，外层查询和子查询对应的记录的 id 列的值都是一样的，则说明查询优化器将子查询转换为了连接查询。

**对于包含 UNION 子句的查询**，每个 SELECT 关键字也都会对应一个唯一的 id 值。但是和包含子查询的查询还是有区别的，例如：

```mysql
-- EXPLAIN 语句
EXPLAIN SELECT * FROM s1  UNION SELECT * FROM s2;

-- EXPLAIN 语句结果集
+----+-------------+----------+------------+------+---------------+-----+---------+-----+------+----------+-----------------+
| id | select_type | table    | partitions | type | possible_keys | key | key_len | ref | rows | filtered | Extra           |
+----+-------------+----------+------------+------+---------------+-----+---------+-----+------+----------+-----------------+
| 1  | PRIMARY     | s1       | NULL       | ALL  | NULL          |NULL | NULL    |NULL | 9688 |   100.00 | NULL            |
+----+-------------+----------+------------+------+---------------+-----+---------+-----+------+----------+-----------------+
| 2  | UNION       | s2       | NULL       | ALL  | NULL          |NULL | NULL    |NULL | 9954 |   100.00 | NULL            |
+----+-------------+----------+------------+------+---------------+-----+---------+-----+------+----------+-----------------+
|NULL|UNION RESULT |<union1,2>| NULL       | ALL  | NULL          |NULL | NULL    |NULL | NULL |     NULL | Using temporary |
+----+-------------+----------+------------+------+---------------+-----+---------+-----+------+----------+-----------------+
```

EXPLAIN 语句结果集中第三条记录 id 值为 NULL ，这是因为 UNION 关键字会将多个查询的结果集合并起来做去重的操作，而去重操作用到了临时表，这个临时表就是 EXPLAIN 语句结果集的第三条记录的由来。

通过 EXPLAIN 语句结果集我们可知，MySQL 在内部创建了一个名为`<union1, 2>`的临时表（就是执行计划第三条记录的 table 列名称），对应EXPLAIN 语句结果集中的第三条记录。记录的 id 列的值为 NULL 表示这个临时表是为了合并两个查询的结果集而创建的。

那不进行合并结果集后的去重，是否就没有 EXPLAIN 语句结果集中的第三条记录了呢？是的，不要去重，也就是不需要临时表，我们可以使用 UNION ALL 关键字（UNION ALL 只是单纯把多个查询结果集中的记录合并成一个返回给用户）来达成这一效果，最终在包含 UNION ALL 子句查询的执行计划中，没有上述 id 列为 NULL 的记录。

<br />

### select_type

---

我们来介绍一下 select_type 字段可以取到的值以及其含义。

* SIMPLE：查询语句中**不包含 UNION 关键字或子查询**，这样的查询都算作是 SIMPLE 类型。

* PRIMARY：对于包含 UNION 或 UNION ALL 或  子查询的大查询，他是由几个小查询组成的，最左边的的那个查询（即执行计划的第一条记录）的 select_type 的值就为 PRIMARY 。

* UNION：对于包含 UNION 或 UNION ALL 或  子查询的大查询，他是由几个小查询组成的，除了最左边的那个小查询（即除了执行计划的第一条记录）以外，其余小查询的 select_type 值就是 UNION 。

* UNION RESULT：UNION 关键字是会对查询的结果集去重的，**MySQL 选择了使用临时表来完成 UNION 查询的去重工作**，针对该临时表的小查询对应执行计划记录的 select_type 值就是 UNION RESULT。

  ```mysql
  --  查询语句
  EXPLAIN SELECT * FROM s1 UNION SELECT * FROM s2;
  
  -- 查询结果
  -- <union1,2> 就是临时表，其 select_type 的值为 UNION RESULT
  +----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
  | id | select_type  | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra           |
  +----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
  |  1 | PRIMARY      | s1         | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 9688 |   100.00 | NULL            |
  |  2 | UNION        | s2         | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 9954 |   100.00 | NULL            |
  |NULL| UNION RESULT | <union1,2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL | NULL |     NULL | Using temporary |
  +----+--------------+------------+------------+------+---------------+------+---------+------+------+----------+-----------------+
  ```

* SUBQUERY：若包含子查询的查询语句不能转换为对应 semi-join 的形式，且该子查询是不相关子查询，同时，查询优化器决定采用将该子查询物化的方案来执行该子查询，这时，该子查询的第一个 SELECT 关键字代表的查询，其 select_type 的值就为 SUBQUERY 。例如：

  ```mysql
  -- 查询语句
  -- 该查询语句外层查询的 WHERE 条件中有其他搜索条件与 IN 子查询组成的布尔表达式使用 OR 连接起来，说明其子查询无法转为 semi-join 
  -- 明显的，该子查询是不相关子查询
  EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2) OR key3 = 'a';
  
  -- 查询结果
  -- 可以看到，查询语句子查询的 select_type 是 SUBQUERY
  -- 由于 select_type 为 SUBQUERY 的子查询会被物化，所以其只需要执行一遍
  +----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
  | id | select_type | table | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra       |
  +----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
  |  1 | PRIMARY     | s1    | NULL       | ALL   | idx_key3      | NULL     | NULL    | NULL | 9688 |   100.00 | Using where |
  |  2 | SUBQUERY    | s2    | NULL       | index | idx_key1      | idx_key1 | 303     | NULL | 9954 |   100.00 | Using index |
  +----+-------------+-------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
  
  ```
  
* DEPENDENT SUBQUERY：若包含子查询的查询语句不能转换为对应 semi-join 的形式，且该子查询是相关子查询，则该子查询的第一个 SELECT 关键字代表的查询，其 select_type 的值就是 DEPENDENT SUBQUERY 。例如：

  ```mysql
  -- 查询语句
  -- 该查询语句外层查询的 WHERE 条件中有其他搜索条件与 IN 子查询组成的布尔表达式使用 OR 连接起来，说明其子查询无法转为 semi-join 
  -- 明显的，该子查询是相关子查询
  EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2 WHERE s1.key2 = s2.key2) OR key3 = 'a';
  
  -- 查询结果
  -- 可以看到，查询语句子查询的第一个 SELECT 关键字代表的查询，其 select_type 的值是 DEPENDENT SUBQUERY
  +----+--------------------+-------+------------+------+---------------+----------+---------+--- -+------+----------+-------------+
  | id |    select_type     | table | partitions | type | possible_keys | key      | key_len | ref | rows | filtered | Extra       |
  +----+--------------------+-------+------------+------+---------------+----------+---------+--- -+------+----------+-------------+
  |  1 | PRIMARY            | s1    | NULL       | ALL  | idx_key3      | NULL     | NULL    |NULL | 9688 |   100.00 | Using where |
  |  2 | DEPENDENT SUBQUERY | s2    | NULL       | ref  | ...           | idx_key2 | 5       |...  |  1   |   10.00  | Using where |
  +----+--------------------+-------+------------+------+---------------+----------+---------+-----+-----+----------+--------------+
  ```

  **注意：select_type 为 DEPENDENT SUBQUERY 的查询可能会被执行多次。**

* DEPENDENT UNION：在包含 UNION 或 UNION ALL 的大查询中，若各个小查询都依赖于外层查询，则除了最左边的那个小查询，其余的小查询的 select_type 值为 DEPENDENT UNION 。例如：

  ```mysql
  -- 查询语句
  EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2 WHERE key1 = 'a' UNION SELECT key1 FROM s1 WHERE key1 = 'b');
  
  -- 查询结果
  +----+--------------------+----------+------------+------+---------------+----------+---------+--- -+------+----------+-------------+
  | id |    select_type     | table    | partitions | type | possible_keys | key      | key_len | ref | rows | filtered | Extra       |
  +----+--------------------+----------+------------+------+---------------+----------+---------+--- -+------+----------+-------------+
  |  1 | PRIMARY            | s1       | NULL       | ALL  | NULL          | NULL     | NULL    |NULL | 9688 |   100.00 | Using where |
  |  2 | DEPENDENT SUBQUERY | s2       | NULL       | ref  | idx_key1      | idx_key1 | 303     |const|   12 |   100.00 | ...         |
  |  3 | DEPENDENT UNION    | s1       | NULL       | ref  | idx_key1      | idx_key1 | 303     |const|    8 |   100.00 | ...         |
  |NULL| UNION RESULT       |<union2,3>| NULL       | ALL  | NULL          | NULL     | NULL    |NULL | NULL |     NULL | ...         |
  +----+--------------------+----------+------------+------+---------------+----------+---------+--- -+------+----------+-------------+
  ```

  这里我不建议用原因推结果，而是建议结果推原因，因为查询语句的子查询怎么看都不像是相关子查询。我们这时候根据查询结果反推，有结论：MySQL认为查询语句的子查询，其中的两个小查询都是相关子查询（我也不知道MySQL为什么这么认为）。

  这样，就能解释查询`SELECT key1 FROM s2 WHERE key1 = 'a'`的 select_type 值为什么为 DEPENDENT SUBQUERY ，而查询`SELECT key1 FROM s1 WHERE key1 = 'b'`的 select_type 值为什么为 DEPENDENT UNION 了。

* DERIVED： 对于采用物化方式执行的，包含派生表的查询，该派生表对应查询的 select_type 的值就是 DERIVED 。例如：

  ```mysql
  -- 查询语句
  EXPLAIN SELECT * FROM (SELECT key1, count(*) as c FROM s1 GROUP BY key1) AS derived_s1 where c > 1;
  
  -- 查询结果
  +----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
  | id | select_type | table      | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra       |
  +----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
  |  1 | PRIMARY     | <derived2> | NULL       | ALL   | NULL          | NULL     | NULL    | NULL | 9688 |    33.33 | Using where |
  |  2 | DERIVED     | s1         | NULL       | index | idx_key1      | idx_key1 | 303     | NULL | 9688 |   100.00 | Using index |
  +----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
  ```

  **注意到，id 为 1 的记录代表的外层查询，它的 table 列显示的是 <derived2>，表明该查询是针对将派生表物化之后的表进行查询的。**若派生表可以通过和外层查询合并的方式执行，执行计划会与上述查询结果大有不同。

* MATERIALIZED：查询优化器在执行包含子查询的语句时，若选择将子查询物化之后与外层查询进行连接查询，该子查询对应的 select_type 的值就是 MATERIALIZED 。例如：

  ```mysql
  -- 查询语句
  EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2);
  
  -- 查询结果
  +----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
  | id | select_type | table      | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra       |
  +----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
  |  1 |SIMPLE       |s1          | NULL       |ALL    | idx_key1      | NULL     | NULL    | NULL | 9688 |   100.00 | Using where |
  |  1 |SIMPLE       |<subquery2> | NULL       |eq_ref | <auto_key>    | ...      | 303     | ...  |    1 |   100.00 | NULL        |
  |  2 |MATERIALIZED |s2          | NULL       |index  | idx_key1      | idx_key1 | 303     | NULL | 9954 |   100.00 | Using index |
  +----+-------------+------------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
  ```

  id 为 2 对应的子查询执行物化后产生的物化表对应的记录就是 table 列值为 <subquery2> 的记录。第一条记录和第二条记录的 id 值是一样的，表明其对应的两个表进行了连接查询。

* UNCACHEABLE SUBQUERY：不常用，不多介绍。

* UNCACHEABLE UNION：不常用，不多介绍。

<br />

### partitions

---





















