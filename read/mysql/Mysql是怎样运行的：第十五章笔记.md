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

## 执行计划输出中各列详解（上）

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

-- EXPLAIN 语句结果集（即执行计划）
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

* PRIMARY：对于包含 UNION 或 UNION ALL 或  子查询 的大查询，他是由几个小查询组成的，最左边的的那个查询（即执行计划的第一条记录）的 select_type 的值就为 PRIMARY 。

* UNION：对于包含 UNION 或 UNION ALL 或  子查询的大查询，他是由几个小查询组成的，除了最左边的那个小查询（即除了执行计划的第一条记录）以外，其余小查询的 select_type 值就是 UNION 。

* UNION RESULT：UNION 关键字是会对查询的结果集去重的，**MySQL 选择了使用临时表来完成 UNION 查询的去重工作**，针对该临时表的小查询对应执行计划记录的 select_type 值就是 UNION RESULT。

  ```mysql
  --  EXPLAIN 语句
  EXPLAIN SELECT * FROM s1 UNION SELECT * FROM s2;
  
  -- 执行计划
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
  -- EXPLAIN 语句
  -- 该查询语句外层查询的 WHERE 条件中有其他搜索条件与 IN 子查询组成的布尔表达式使用 OR 连接起来，说明其子查询无法转为 semi-join 
  -- 明显的，该子查询是不相关子查询
  EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2) OR key3 = 'a';
  
  -- 执行计划
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
  -- EXPLAIN 语句
  -- 该查询语句外层查询的 WHERE 条件中有其他搜索条件与 IN 子查询组成的布尔表达式使用 OR 连接起来，说明其子查询无法转为 semi-join 
  -- 明显的，该子查询是相关子查询
  EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2 WHERE s1.key2 = s2.key2) OR key3 = 'a';
  
  -- 执行计划
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
  -- EXPLAIN 语句
  EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2 WHERE key1 = 'a' UNION SELECT key1 FROM s1 WHERE key1 = 'b');
  
  -- 执行计划
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

  这样，就能解释查询`SELECT key1 FROM s2 WHERE key1 = 'a'`的 select_type 值为什么为 DEPENDENT SUBQUERY ，而查询`SELECT key1 FROM s1 WHERE key1 = 'b'`的 select_type 值为什么为 DEPENDENT UNION 了。同时，我们的思维也要更加灵活，不能局限于用定义证结果，我们也要能通过结果了解信息。

* DERIVED：对于采用物化方式执行的，包含派生表的查询，该派生表对应查询的 select_type 的值就是 DERIVED 。例如：

  ```mysql
  -- EXPLAIN 语句
  EXPLAIN SELECT * FROM (SELECT key1, count(*) as c FROM s1 GROUP BY key1) AS derived_s1 where c > 1;
  
  -- 执行计划
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
  -- EXPLAIN 语句
  EXPLAIN SELECT * FROM s1 WHERE key1 IN (SELECT key1 FROM s2);
  
  -- 执行计划
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

由于我们未介绍过分区的概念，该字段不多介绍。一般而言，查询语句的执行计划，其中记录的 partitions 列的值均为 null 。

<br />

### type

---

**执行计划的一条记录就代表着 MySQL 对某个表执行查询时的访问方法，而 type 字段，就代表了这个访问方法是什么。**

在前面的章节中，我们只介绍了对使用 InnoDB 存储引擎的表进行单表访问的一些访问方法，完整的访问方法却远远不止这些，下面我们来一一介绍它们：

* system：当表中只有一条记录并且该表使用的存储引擎的统计数据是精确的（例如 MyISAM、Memory ），那么该表的访问方法就是 system 。

* const：当我们根据主键或者唯一二级索引列与常数进行等值匹配时，对单表的访问方法就是 const 。

* eq_ref：**在连接查询时**，若被驱动表是通过主键或者唯一二级索引列等值匹配的方式进行访问的（若该主键或者唯一二级索引是联合索引的话，所有的索引列都必须进行等值比较），则**对该被驱动表的访问方法**就是 eq_ref 。

* ref：当通过普通二级索引列与常量进行等值匹配来查询某个表，这时对该表的访问方法就**可能**是 ref 。

* fulltext：全文索引，暂时跳过不细讲。

* ref_or_null：当对普通二级索引进行等值匹配查询，且该索引列的值也可以是 NULL 值时，对该表的访问方法就**可能**是 ref_or_null 。

* index_merge：一般而言，对某个表的查询只能使用到一个索引，但是在特殊的场景下，也是可以使用 Intersection、Unioin、Sort-Union 这三种索引合并的方式来执行查询的（第十章笔记有索引合并的详细介绍）。当使用索引合并的方式来查询某个表，这时对该表的访问方法就是 index_merge 。

* unique_subquery：unique_subquery 访问方法针对一些包含 IN 子查询的查询语句，若查询优化器决定将 IN 子查询转换为 EXISTS 子查询，且子查询可以使用到**主键**进行等值匹配，那么该子查询对应执行计划中记录的 type 字段的值就是 unique_subquery 。例如：

  ```mysql
  -- EXPLAIN 语句
  EXPLAIN SELECT * FROM s1 WHERE key2 IN (SELECT id FROM s2 where s1.key1 = s2.key1) OR key3 = 'a';
  
  -- 执行计划
  +----+--------------------+-------+------------+-----------------+---------------+--------+-------+-----+------+----------+-----+
  | id | select_type        | table | partitions | type            | possible_keys |key     |key_len| ref | rows | filtered |Extra|
  +----+--------------------+-------+------------+-----------------+---------------+--------+-------+-----+------+----------+-----+
  |  1 | PRIMARY            | s1    | NULL       | ALL             | idx_key3      |NULL    | NULL  |NULL | 9688 |   100.00 | ... |
  |  2 | DEPENDENT SUBQUERY | s2    | NULL       | unique_subquery | ...           |PRIMARY | 4     |func |    1 |    10.00 | ... |
  +----+--------------------+-------+------------+-----------------+---------------+--------+-------+-----+------+----------+-----+
  ```

  **注意，“且子查询可以使用到主键进行等值匹配”并不是说子查询的查询条件中必须出现主键进行等值匹配。**我们从上例中的执行计划中可以看出，其第二条记录的 type 值就是 unique_subquery ，说明在执行子查询时会使用到 id 列的索引。

* index_subquery：index_subquery 访问方法针对一些包含 IN 子查询的查询语句，若查询优化器决定将 IN 子查询转换为 EXISTS 子查询，且子查询可以使用到**普通的索引**进行等值匹配，那么该子查询对应执行计划中记录的 type 字段的值就是 index_subquery 。例如：

  ```mysql
  -- EXPLAIN 语句
  EXPLAIN SELECT * FROM s1 WHERE common_field IN (SELECT key3 FROM s2 where s1.key1 = s2.key1) OR key3 = 'a';
  
  -- 执行计划
  +----+--------------------+-------+------------+-----------------+---------------+--------+-------+-----+------+----------+-----+
  | id | select_type        | table | partitions | type            | possible_keys |key     |key_len|ref  | rows | filtered |Extra|
  +----+--------------------+-------+------------+-----------------+---------------+--------+-------+-----+------+----------+-----+
  |  1 | PRIMARY            | s1    | NULL       | ALL             | idx_key3      |NULL    | NULL  |NULL | 9688 |   100.00 | ... |
  |  2 | DEPENDENT SUBQUERY | s2    | NULL       | index_subquery  | ...           |idx_key3| 303   |func |    1 |    10.00 | ... |
  +----+--------------------+-------+------------+-----------------+---------------+--------+-------+-----+------+----------+-----+
  ```

  **注意，“且子查询可以使用到普通的索引进行等值匹配”并不是说子查询的查询条件中必须出现普通的索引进行等值匹配。我们从上例中的执行计划中可以看出，**其第二条记录的 type 值就是 index_subquery ,说明在执行子查询时会使用到 key3 列的索引。

* range: 若使用索引获取某些范围区间的记录，则**可能**使用到 range 访问方法，例如：

  ```mysql
  -- 例1
  -- IN 关键字会构建出许多的单点区间
  -- 单点区间可以被视为特殊的范围区间
  SELECT * FROM s1 WHERE key1 IN ('a', 'b', 'c');
  
  -- 例2
  SELECT * FROM s1 WHERE key1 > 'a' AND key1 < 'b';
  ```

* index：当我们可以使用到索引覆盖，但需要扫描全部的索引记录时，该表的访问方法就是 index 。例如：

  ```mysql
  -- 例1
  -- key_part2 列和 key_part3 列均包含在 idx_key_part 联合索引中
  -- 搜索条件 key_part3 = 'a' 无法使用联合索引 idx_key_part 进行 ref 或 range 方式的访问，只能扫描整个联合索引 idx_key_part 的记录
  -- 综上所述，该查询语句对应查询计划中记录的 type 字段的值为 index 
  SELECT key_part2 FROM s1 WHERE key_part3 = 'a';
  ```

  **注意：对于使用 InnoDB 存储引擎的表而言，二级索引的记录只包含索引列和主键列的值，而聚簇索引中包含用户定义的全部列以及一些隐藏列，所以扫描二级索引的代价比直接全表扫描（即扫描聚簇索引）的代价更低一些。**

* ALL：全表扫描，无需多言。

**上述访问方法从上到下，性能依次降低。**

**除了 ALL 访问方法，其余的访问方法都可以用到索引。**

**除了 index_merge 访问方法，其余的访问方法都最多只能用到一个索引。**

<br />

### possible_keys 和 key

---

**字段 possible_keys **表示在某个查询语句中，对某个表执行单表查询时可能使用到的索引，而**字段 key **表示实际用到的索引。

这里有一个误区，**possible_keys 列中的值并不是越多越好**，因为查询优化器计算查询成本的时间是随着 possible_keys 列含有值数量的增加而增加的，若可以，尽量删除那些用不到的索引。

**特别的，使用 index 访问方法查询某个表时，其对应的执行计划中，记录的 possible_keys 列是空的**，而 key 列仍旧是展示的是实际用到的索引。

<br />

### key_len

---

字段 key_len 表示当优化器决定使用某个索引执行查询时，该索引记录的最大长度。

索引记录的最大长度，由三个部分构成，分别是：

* 对于使用固定长度类型的索引列，它实际占用的存储空间的最大长度就是该固定值。对于指定字符集的变长类型的索引列，例如某个索引列的类型是`VARCHAR(100)`，使用的字符集是 utf8（本书第三章字符集和比较规则中写明了，本书的 utf8 字符集指的是阉割过的 utf8 字符集，即 utf8mb3 ，它只使用 1 ~ 3 个字节表示字符），那么该列实际占用的**最大**存储空间就是：100 * 3 = 300（字节）。
* 若该索引列可以存储 NULL 值，则 key_len 比不可以存储 NULL 值时多 1 个字节。
* 对于变长字段而言，会有额外的 2 个字节的空间用来存储该变长列的实际长度。

前两部分没啥问题，第三部分问题大了，对于 InnoDB 行格式，存储变长字段的实际长度不是可能占用1个字节或者2个字节吗？怎么统统统一成 2 字节了？

首先，执行计划的生成在 MySQL server 层，并不是针对某一存储引擎的功能。

其次，key_len 列设计出来的目的主要是为了让我们区分某个使用联合索引的查询具体用了几个索引列。我们先举一些例子分析 key_len 的值，再来说怎么区分某个使用联合索引的查询具体用了几个索引列。

```mysql
-- 例1 
-- EXPLAIN 语句
-- 其执行计划只有一条记录且对应着表 s1 ，这条记录 key_len 列的值是 4 。因为 id 主键不可以为 NULL ，且其类型是 INT ，占用 4 字节，所以共占用 4 字节
EXPLAIN SELECT * FROM s1 WHERE id = 5;

-- 例2
-- EXPLAIN 语句
-- 其执行计划只有一条记录且对应着表 s1 ，这条记录 key_len 列的值是 5 。因为 key2 列的类型是 INT ，占用 4 字节 ，且其值可以为 NULL （加 1），所以共占用 5 字节
EXPLAIN SELECT * FROM s1 WHERE key2 = 5;

-- 例3
-- EXPLAIN 语句
-- 其执行计划只有一条记录且对应着表 s1 ，这条记录 key_len 列的值是 303 。因为 key1 列的类型是 VARCHAR(100)，上文分析过该列实际占用的最大存储空间是 300 字节，又因为其值可以为 NULL（加 1），且该列是可变长度列（加 2），所以共占用 303 字节
EXPLAIN SELECT * FROM s1 WHERE key1 = 'a';
```

现在，我们来举几个例子来说明怎么区分某个使用联合索引的查询具体用了几个索引列。

```mysql
-- 例1
-- EXPLAIN 语句
EXPLAIN SELECT * FROM s1 WHERE key_part1 = 'a';

-- 执行计划
-- 可以看到执行计划中 key_len 列的值是 303 ，这说明在执行例 1 的查询中，只能用到 idx_key_part 联合索引的一个索引列
+----+-------------+-------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key          | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+--------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | s1    | NULL       | ref  | idx_key_part  | idx_key_part | 303     | const |   12 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+--------------+---------+-------+------+----------+-------+

-- 例2
-- EXPLAIN 语句
EXPLAIN SELECT * FROM s1 WHERE key_part1 = 'a' AND key_part2 = 'b';

-- 执行计划
-- 可以看到执行计划中 key_len 列的值是 606 ，这说明在执行例 2 的查询中，可以用到 idx_key_part 联合索引的两个索引列
+----+-------------+-------+------------+------+---------------+--------------+---------+-------------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key          | key_len | ref         | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+--------------+---------+-------------+------+----------+-------+
|  1 | SIMPLE      | s1    | NULL       | ref  | idx_key_part  | idx_key_part | 606     | const,const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+--------------+---------+-------------+------+----------+-------+
```

<br />

### ref

---

ref 列展示的是使用索引列等值匹配的条件去执行查询时，与索引列作等值匹配的是什么。多说无益，请看下面的一些例子：

```mysql
-- 例1
-- EXPLAIN 语句
-- 其执行计划只有一条记录且对应着表 s1 ，这条记录 ref 列的值是 const ，表明在使用 idx_key1 索引执行查询时，与 key1 列作等值匹配的对象是一个常数
EXPLAIN SELECT * FROM s1 WHERE key1 = 'a';

-- 例2
-- EXPLAIN 语句
-- 其执行计划中有两条记录，表 s1 对应的记录在表 s2 对应的记录之前，表明表 s2 为被驱动表
-- 对于表 s2 的那条记录其 ref 列的值为 xiaohaizi.s1.id ，表明对被驱动表进行访问时会用到主键索引，与 s2.id 列作等值匹配的对象是 xiaohaizi.s1.id 列（这里把数据库名都写出来了）
EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.id = s2.id;

-- 例3
-- EXPLAIN 语句
-- 其执行计划中有两条记录，表 s1 对应的记录在表 s2 对应的记录之前，表明表 s2 为被驱动表
-- 对于表 s2 的那条记录其 ref 列的值为 func ，说明与 s2.key1 列作等值匹配的对象是一个函数
EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s2.key1 = UPPER(s1.key1);
```

<br />

### rows

---

rows 字段的值有两种情况：

* 若查询优化器决定使用全表扫描的方式对某个表执行查询，则执行计划的 rows 列代表预计需要扫描的行数。
* 若查询优化器决定使用索引来执行查询，则执行计划的 rows 列代表预计扫描的索引记录行数。

例如：

```mysql
-- EXPLAIN 语句
-- 其执行计划只有一条记录且对应着表 s1 ，这条记录 rows 列的值是 266 ,表明查询优化器在经过分析使用 idx_key1 进行查询的成本之后，觉得满足 key1 > 'z' 条件的记录只有266条。
EXPLAIN SELECT * FROM s1 WHERE key1 > 'z';
```

<br />

### filtered

---

还记得 Condition Filtering 的含义吗？这是 MySQL 在计算驱动表扇出时采用的一个策略。

如果不记得我们先来温习一下。

计算扇出值情况可以分以下两种：

- 若使用的是全表扫描的方式执行的单表查询，则计算驱动表扇出时需要**猜**满足搜索条件的记录到底有多少条。
- 若使用的是索引执行的单表扫描，则计算驱动表扇出的时候需要**猜**满足除使用到对应索引的搜索条件外的其他搜索条件的记录有多少条。

虽然情况分两种，但有一个共同的特点，就是**猜**。

这个猜的过程被称为 **Condition Filtering** 。这个过程可能会用到索引、统计数据或启发式规则（即根据以往经验指定的一些规则） 。

Condition Filtering 被应用在计算 filtered 列的值中，表现如下：

```mysql
-- EXPLAIN 语句
EXPLAIN SELECT * FROM s1 WHERE key1 > 'z' AND common_field = 'a';

-- 执行计划
-- 执行计划表明例 1 查询使用idx_key1索引来执行查询，从rows列可以看出满足key1 > 'z'的记录有266条。
-- 满足其余搜索条件（即搜索条件 common_field = 'a' ）的记录数（猜测值）占 266 的百分比就是 filtered 列的值，从执行计划中可以看出，查询对应执行计划记录的 filtered 列的值为 10%
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | s1    | NULL       |range | idx_key1      | idx_key1 | 303     | NULL  |  266 |    10.00 | ...   |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------+
```

上述 EXPLAIN 语句是对于单表查询而言的，对于单表查询，filtered 列的值没有什么意义，我们更关注在连接查询中驱动表对应执行计划记录的 filtered 列值。例如：

```mysql
-- EXPLAIN 语句
EXPLAIN SELECT * FROM s1 INNER JOIN s2 ON s1.key1 = s2.key1 WHERE s1.common_field = 'a';

-- 执行计划
+----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref               | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+-------------+
|  1 | SIMPLE      | s1    | NULL       | ALL  | idx_key1      | NULL     | NULL    | NULL              | 9688 |    10.00 | Using where |
|  1 | SIMPLE      | s2    | NULL       | ref  | idx_key1      | idx_key1 | 303     | xiaohaizi.s1.key1 |    1 |   100.00 | NULL        |
+----+-------------+-------+------------+------+---------------+----------+---------+-------------------+------+----------+-------------+
```

该查询语句的执行计划表明，查询优化器打算把表 s1 当作驱动表，表 s2 当作被驱动表。我们可以看到表 s1 执行计划的 rows 列值为 9688 ，filtered 列值为 10.00 。这意味着驱动表 s1 的扇出值就是`9688 × 10.00% = 968.8`，表明还需要对被驱动表执行大约 968 次查询。















