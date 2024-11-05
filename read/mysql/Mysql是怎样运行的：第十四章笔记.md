# Mysql是怎样运行的：第十四章笔记

---

## 前言

---

根据一定的规则，将执行起来十分耗费性能的语句重写成某种可以以比较高效率执行的形式，这个过程便是**查询重写。**

本章将围绕着查询重写中较为重要的一些**重写规则**展开论述。

<br />

## 条件化简

---

查询语句的搜索条件本质上是一个表达式，若该表达式繁杂抑或是低效，MySQL 的查询优化器就会为我们简化这些表达式。

化简的方式如下（a，b，c等字母均代表列名）：

* 移除不必要的括号。例如

  ```mysql
  -- 化简前
  ((a = 5 AND b = c) OR ((a > c) AND (c < 5)))
  
  -- 化简后
  (a = 5 and b = c) OR (a > c AND c < 5)
  ```

* 常量传递（Constant_Propagation）。即表达式是某个列 a 和某个常量做等值匹配，并且该表达式和其他涉及列 a 的表达式用关键字 AND 连接，那么就可以将其他涉及列 a 的表达式中 a 的值替换成做等值匹配的常量值。例如：

  ```mysql
  -- 化简前
  a = 5 AND b > a
  
  -- 化简后
  a = 5 AND b > 5
  ```

* 等值传递（Equality_Propagation）。例如：

  ```mysql
  -- 化简前
  a = b and b = c and c = 5
  
  -- 化简后
  a = 5 and b = 5 and c = 5
  ```

* 移除没用的条件（Trivial_Condition_Removal）。明显永远为 TRUE 或 FALSE 的表达式，优化器会移除他们。例如：

  ```mysql
  -- 化简前
  (a < 1 and b = b) OR (a = 6 OR 5 != 5)
  
  -- 化简后
  a < 1 OR a = 6
  ```

* 表达式计算。即查询执行之前，若表达式中只包含常量，则它的值会被先计算，例如：

  ```mysql
  -- 化简前（注意到表达式右边只包含常量）
  a = 5 + 1
  
  -- 化简后
  a = 6
  ```

  但是，如果列并不是以单独的形式作为表达式的操作数，那么优化器是不会尝试对这些表达式进行化简的。例如：

  ```mysql
  -- 列出现在函数中
  ABS(a) > 5
  
  -- 列出现在复杂的表达式中
  -a < -8
  ```

  我们知道，只有搜索条件中索引列和常数使用某些运算符连接起来才可能使用到索引，所以如果可以，**最好让索引列以单独的形式出现在表达式中。**
  
* HAVING 子句和 WHERE 子句的合并。即若查询语句中没有出现聚合函数（例如 SUM、MAX等）以及 GROUP BY 子句，那么查询优化器就会将 HAVING 子句和 WHERE 子句合并。

* MySQL 认为以下两种查询速度特别快：

  * 查询的表中无记录或只有一条记录。
  * 使用主键等值匹配或者唯一二级索引列等值匹配作为搜索条件来查询某个表。

  这里的问题是，我们怎么在查询前知道查询的表中无记录或只有一条记录呢？当然是通过统计数据只晓的。不过我们说过InnoDB的统计数据数据不准确，所以这一条不能用于使用InnoDB作为存储引擎的表，只能适用于使用Memory或者MyISAM存储引擎的表。

  通过这两种方式查询的表称之为**常量表（Constant Tables）**，优化器在分析一个查询语句时，首先执行常量表查询，然后把查询中涉及到该表的条件全部替换成常数，最后再分析其余表的查询成本。例如：

  ```mysql
  -- 明显的，table1 表便是常量表
  SELECT * FROM table1 INNER JOIN table2
      ON table1.column1 = table2.column2 
      WHERE table1.primary_key = 1;
  ```

  在分析 table2 表的查询成本前，就会对 table1 执行查询，把其中涉及到 table1 表的条件全部替换，像这样：

  ```mysql
  SELECT 
  	table1表记录的各个字段的常量值,
  	table2.* FROM table1 INNER JOIN table2 ON table1表column1列的常量值 = table2.column2;
  ```

<br />

## 外连接消除

---

左、右外连接驱动表和被驱动表是固定的，但是内连接却与之相反，这也就是说，内连接可能可以通过优化表的连接顺序来降低整体的查询成本，但外连接不行。

那么有没有一种转换，可以把外连接转换成内连接从而使得外连接也可以通过优化表的连接顺序来降低整体的查询成本呢？

有的，首先我们看左、右外连接，查询优化器如何对待它们。

由于左、右外连接只是在驱动表选取上不同，其余方面相同，所以查询优化器会首先把右外连接转换成左外连接查询，这样呢，左、右外连接就可以统一了，我们就可以只探寻，外连接和内连接是如何转换的了。

我们知道对于关键字 ON ，内、外连接使用是效果不同的，但是对于关键字 WHERE ，内、外连接的效果都一致，那就是**凡是不符合WHERE子句中条件的记录都不会参与连接。**

对于外连接的驱动表的记录来说，如果无法在被驱动表中找到匹配ON子句中的过滤条件的记录，那么该记录仍然会被加入到结果集中，且被驱动表相关列的值为 NULL ；而内连接的驱动表的记录如果无法在被驱动表中找到匹配ON子句中的过滤条件的记录，那么该记录会被舍弃。

救赎之道就在其中，NULL ！我们只需要将被驱动表相关列的值为 NULL 的记录在外连接结果集中排除，就可以达成外连接和内连接的相互转换！

如何排除，当然是使用 WHERE 关键字了。这一种在外连接查询中，指定 WHERE 子句中包含被驱动表中的列不为 NULL 值的条件称之为**空值拒绝（Reject-NULL）**。

那么如何用关键字 WHERE 指定包含被驱动表中的列不为 NULL 值呢？直接间接的都行，例如：

```mysql
-- 直接
WHERE 被驱动表列n1 IS NOT NULL;

-- 间接，假设 被驱动表列n1 数据类型是 INT
WHERE 被驱动表列n1 = 2
```

在被驱动表的 WHERE 子句符合空值拒绝的条件后，外连接和内连接可以相互转换。查询优化器可以通过评估表的不同连接顺序的成本，选出成本最低的那种连接顺序来执行查询。

<br />

## 子查询优化

---

现给出两个表 t1 和 t2，他们的结构如下：

```mysql
CREATE TABLE t1 (
    m1 int, 
    n1 char(1)
) Engine=InnoDB, CHARSET=utf8;

CREATE TABLE t2 (
    m2 int, 
    n2 char(1)
) Engine=InnoDB, CHARSET=utf8;

```

出现在某个查询语句的某个位置中的查询被称为**子查询**，而这个某个查询，就是**外层查询。**子查询可以在对应外层查询的各种位置出现，例如：

* SELECT 子句中。

  ```mysql
  SELECT (SELECT m1 FROM t1 LIMIT 1);
  ```

* FROM 子句中。

  ```mysql
  -- 放在 FROM 子句中的子查询，其结果集组成的表称之为派生表，下面的 t 表便是派生表
  SELECT m, n FROM (SELECT m2 + 1 AS m, n2 AS n FROM t2 WHERE m2 > 2) AS t;
  ```

* WHERE 或 ON 子句中。

  ```mysql
  SELECT * FROM t1 WHERE m1 IN (SELECT m2 FROM t2);
  ```

* ORDER BY 子句中。

* GROUP BY 子句中。

<br />

### 按返回的结果集区分子查询

---

按返回的不同结果集类型，把子查询分为不同的类型，如下：

* 标量子查询。即只返回一个单一值的子查询，这个单一的值，也被称为**标量**。

  ```mysql
  -- 例1：子查询返回 t1 表一条记录的 m1 列的值
  SELECT (SELECT m1 FROM t1 LIMIT 1);
  
  -- 例2：子查询返回 t2 表 m2 列的最小值
  SELECT * FROM t1 WHERE m1 = (SELECT MIN(m2) FROM t2);
  ```

* 行子查询。即只返回一条记录的子查询，但这条记录包括多个列。

  ```mysql
  -- 例：子查询返回 t2 表一条记录的 m1 和 n1 列的值
  SELECT * FROM t1 WHERE (m1, n1) = (SELECT m2, n2 FROM t2 LIMIT 1);
  ```

* 列子查询。即返回一个列多条记录的子查询，不能只返回一条记录，否则是标量子查询。

  ```mysql
  -- 例：子查询返回 t2 表多条记录的 m2 列的值
  SELECT * FROM t1 WHERE m1 IN (SELECT m2 FROM t2);
  ```

* 表子查询。即返回多条记录，每条记录包含多个列的子查询。

  ```mysql
  -- 例：子查询返回 t2 表的多条记录，每条记录包含 m2 和 n2 列的值
  SELECT * FROM t1 WHERE (m1, n1) IN (SELECT m2, n2 FROM t2);

<br />

### 按与外层查询关系来区分子查询

---

子查询不依赖于外层查询的值就可以查询出结果，这样的子查询被称为**不相关子查询**。而子查询依赖于外层查询的值才可以查询出结果，这样的子查询被称为**相关子查询**。

<br />

### 子查询在布尔表达式中的使用

---

我们总结一下在布尔表达式中使用子查询的场景。

* 使用`=`、`>`、`<`、`>=`、`<=`、`<>`、`!=`、`<=>`作为布尔表达式的操作符。我们给这些操作符一个名字，为 `comparison_operator`，那么，子查询组成的布尔表达式如下：

  ```mysql
  -- 操作数可以为某个列名、一个常量或一个更复杂的表达式，甚至是另一个子查询
  操作数 comparison_operator (子查询)
  ```

  这里的子查询必须是标量子查询或行子查询，也就是子查询的结果只能返回一个单一的值或者只能是一条记录。例如：

  ```mysql
  -- 标量子查询
  SELECT * FROM t1 WHERE m1 < (SELECT MIN(m2) FROM t2);
  
  -- 行子查询
  SELECT * FROM t1 WHERE (m1, n1) = (SELECT m2, n2 FROM t2 LIMIT 1);
  ```

* [NOT] IN/ANY/SOME/ALL子查询。这是对于集合而言的，即 MySQL 通过如下的语法来支持操作数和一个集合组成一个布尔表达式：

  * `IN`或者`NOT IN`。用来判断某个操作数在不在由子查询结果集组成的集合中。

    ```mysql
    -- 语法
    操作数 [NOT] IN (子查询)
    
    -- 例子
    SELECT * FROM t1 WHERE (m1, n2) IN (SELECT m2, n2 FROM t2);
    ```

  * `ANY/SOME`（`ANY`和`SOME`是同义词）。即只要子查询结果集中存在某个值和给定的操作数做比较结果为`TRUE`，那么整个表达式的结果就为`TRUE`。

    ```mysql
    -- 例子，下面两句 SQL 等价
    SELECT * FROM t1 WHERE m1 > ANY(SELECT m2 FROM t2);
    SELECT * FROM t1 WHERE m1 > (SELECT MIN(m2) FROM t2);
    ```

    **注意：`= ANY`相当于`IN`。**

  * ALL 。即子查询结果集中所有的值和给定的操作数做比较结果为`TRUE`，那么整个表达式的结果就为`TRUE`。

    ```mysql
    -- 例子，下面两句 SQL 等价
    SELECT * FROM t1 WHERE m1 > ALL(SELECT m2 FROM t2);
    SELECT * FROM t1 WHERE m1 > (SELECT MAX(m2) FROM t2);
    ```

  * EXISTS 子查询。即有的时候我们仅仅需要判断子查询的结果集中是否有记录，而不在乎它的记录具体是什么，可以使用把`EXISTS`或者`NOT EXISTS`放在子查询语句前面。

    ```mysql
    -- 例子
    SELECT * FROM t1 WHERE EXISTS (SELECT 1 FROM t2);
    ```

    我们只关心子查询`(SELECT 1 FROM t2)`的结果集中是否有记录，如果存在记录则整个`EXISTS`表达式的结果就为`TRUE`。

<br />

### 子查询语法注意事项

---

话不多说，注意事项如下：

* 子查询必须用小括号括起来。

* 在 SELECT 子句中的子查询必须是标量子查询。

  ```mysql
  -- 例子：非法子查询，子查询结果集中有多个列或者多个行
  SELECT (SELECT m1, n1 FROM t1);
  ```

* 想要得到标量子查询或者行子查询，但又不能保证子查询的结果集只有一条记录时，应该使用`LIMIT 1`语句来限制记录数量。

* 对于`[NOT] IN/ANY/SOME/ALL`子查询来说，子查询中不允许有`LIMIT`语句（硬性规定）。

  ```mysql
  -- 例子：非法子查询
  SELECT * FROM t1 WHERE m1 IN (SELECT * FROM t2 LIMIT 2);
  ```

  因为`[NOT] IN/ANY/SOME/ALL`子查询不支持`LIMIT`语句，所以子查询中的一些语句也就显得十分的多余。

  * `ORDER BY`子句。

    ```mysql
    -- 例子
    -- 下面 SQL 中的 ORDER BY 子句显得画蛇添足，因为子查询结果集里的值排序不怎么重要
    SELECT * FROM t1 WHERE m1 IN (SELECT m2 FROM t2 ORDER BY m2);
    ```

  * `DISTINCT`语句 。

    ```mysql
    -- 例子
    -- 子查询结果集是否去重对下面的 SQL 不重要
    SELECT * FROM t1 WHERE m1 IN (SELECT DISTINCT m2 FROM t2);
    ```

  * 没有聚集函数以及`HAVING`子句的`GROUP BY`子句。

    ```mysql
    -- 例子
    -- 没有聚集函数以及 HAVING 子句，GROUP BY 子句就是个摆设
    SELECT * FROM t1 WHERE m1 IN (SELECT m2 FROM t2 GROUP BY m2);
    ```

  对于这些冗余的语句，查询优化器在一开始就会把它们给干掉。

* 不允许在一条语句中增删改某个表的记录时，同时还对该表进行子查询。

  ```mysql
  -- 例子：非法子查询
  DELETE FROM t1 WHERE m1 < (SELECT MAX(m1) FROM t1);
  ```

<br />

### 子查询在 MySQL 中是怎么执行的

---

假设存在两个表 s1 和 s2 ，他们的表结构相同，各有 10000 条记录，除 id 列外其余的列用随机值填充。表结构如下：

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

下面介绍的 MySQL 如何优化子查询执行方式的方式，都是基于 MySQL5.7 这个版本讲解的。

<br />

#### 标量子查询、行子查询的执行方式

---

以下两种场景经常使用到标量子查询或者行子查询：

* 在 SELECT 子句中的子查询必须是标量子查询。
* 子查询使用`=`、`>`、`<`、`>=`、`<=`、`<>`、`!=`、`<=>`等操作符和某个操作数组成一个布尔表达式，这样的子查询必须是标量子查询或者行子查询。

对于以上两种情况，**不相关标量子查询或行子查询**，他们的执行方式是简单的，例如：

```mysql
SELECT * FROM s1 
WHERE key1 = (SELECT common_field FROM s2 WHERE key3 = 'a' LIMIT 1);
```

执行过程如下：

* 先单独执行`(SELECT common_field FROM s2 WHERE key3 = 'a' LIMIT 1)`子查询。
* 将上一步子查询得到的结果当作外层查询的参数再执行外层查询`SELECT * FROM s1 WHERE key1 = ...`。

即对于包含**不相关的标量子查询或行子查询**的查询语句来说，MySQL 会分别独立的执行外层查询和子查询，相当于进行两个单表查询。

对于以上两种情况，**相关的标量子查询或行子查询**，他们的执行方式也是简单的，例如：

```mysql
SELECT * FROM s1 
WHERE key1 = (
    SELECT common_field FROM s2 
    WHERE s1.key3 = s2.key3 LIMIT 1
);
```

执行过程如下：

1. 先从外层查询中获取一条记录，即先从 s1 表中获取一条记录。
2. 从上一步中获取的那条记录中找出子查询中涉及到的值，即从 s1 表中获取的那条记录中找出`s1.key3`列的值，然后执行子查询。
3. 最后根据子查询的查询结果来检测外层查询 WHERE 子句的条件是否成立，如果成立，就把外层查询的那条记录加入到结果集，否则丢弃。
4. 再次执行第一步，获取第二条外层查询中的记录，依次类推。

<br />

#### IN 子查询的优化

---

对于不相关的 IN 子查询，例如：

```mysql
SELECT * FROM s1 
WHERE key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a');
```

如果以标量子查询或行子查询的执行方式来看待 IN 子查询，即将子查询和外层查询分别看成两个单独的单表查询，在子查询结果集中的记录数较少的时候，IN 子查询的效率，还是可以的。但是如果子查询结果集中的记录太多，就会导致以下问题：

* 结果集记录太多，内存放不下。

* 外层查询的 IN 子句中参数特别多，会导致无法有效的使用索引，只能对外层查询进行全表扫描。并且在对外层查询执行全表扫描时，检测一条记录是否符合和 IN 子句中的参数匹配，花费的时间太长。

  ```mysql
  -- 例子
  -- 每条记录需要判断一下它的 column 列是否符合 column = a OR column = b OR column = c OR ... ，这样的性能消耗是十分巨大的，并且外层查询无法有效的使用索引。
  SELECT * FROM tbl_name WHERE column IN (a, b, c ..., ...);
  ```

MySQL 解决这个问题的方案是：**不直接将不相关子查询的结果集当作外层查询的参数，而是将该结果集写入一个临时表里。**这个过程如下：

* 该临时表的列就是子查询结果集中的列。
* 写入临时表的记录会被去重（可以使得临时表更小，更省地方）。

```
问：临时表如何对记录进行去重？
答：临时表也是个表，只要为表中记录的所有列建立主键或者唯一索引即可。
```

**当子查询结果集较小的时候**，MySQL 会为它建立基于内存的，使用 Memory 存储引擎的临时表，并且会为该表建立哈希索引。IN 语句的本质就是判断某个操作数在不在某个集合里，如果集合中的数据建立了哈希索引，那么这个匹配的过程就会十分迅速。 

 **当子查询结果集较大的时候**，超过了系统变量`tmp_table_size`或者`max_heap_table_size`的值，临时表会转而使用基于磁盘的存储引擎来保存结果集中的记录，索引类型也对应转变为 B+ 树索引。

这种将子查询结果集中的记录保存到临时表的过程称之为**物化（英文名：Materialize）**。我们这里把存储子查询结果集的临时表称之为**物化表**。正因为物化表中的记录都建立了索引（基于内存的物化表有哈希索引，基于磁盘的物化表有B+树索引），通过索引执行 IN 语句判断某个操作数在不在子查询的结果集中就变得非常快，提升了子查询语句的性能。

<br />

#### 物化表转连接

---

对于查询语句：

```mysql
SELECT * FROM s1 
WHERE key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a');
```

假设将子查询进行物化后，子查询物化表的名称为 materialized_table ，该物化表存储的子查询结果集的列为 m_val ，那么，我们可以从两个不同的角度看待上述查询。

* 从表 s1 的角度，即对表 s1 的每条记录，若该记录的 key1 列的值在表 materialized_table 中，那么该记录会被加入最终的结果集。
* 从表 materialized_table 的角度，即对于表 materialized_table 的每个值，若能在 s1 表中找到对应的 key1 列的值与该值相等的记录，则将对应记录加入到最终的结果集中。

将上述两种角度整合，不就是表 s1 和表 materialized_table 的内连接吗？即查询语句等价于下面的 SQL 语句：

```mysql
SELECT s1.* FROM s1 INNER JOIN materialized_table ON key1 = m_val;
```

这样的话，查询优化器就可以评估不同连接顺序需要的成本，从而选取成本最低的那种查询方式执行查询了，我们简单分析一下吧。

* 表 s1 为驱动表，查询成本由以下几个部分组成：
  * 物化子查询时需要的成本。
  * 扫描表 s1 的成本。
  * 表 s1 中记录的数量 * 通过`m_val = xxx`对表 materialized_table 进行单表访问的成本（这里`m_val = xxx`中的 xxx 即表 s1 中每条记录的 key1 列的值，其次因为物化表中记录不重复且物化表中的列被建立的索引，这个步骤是非常快的）。
* 表 materialized_table 为驱动表，查询成本由以下几个部分组成：
  * 物化子查询时需要的成本。
  * 扫描物化表时的成本
  * 物化表中的记录数量 * 通过`key1 = xxx`对表 s1 进行单表访问的成本（这里`key1 = xxx`中的 xxx 即表 materialized_table 中每条记录的 m_val 列的值，其次因为表 s1 的 key1 列上存在索引，所以这个步骤也是非常快的）。

MySQL 查询优化器会通过运算来选择上述成本更低的方案来执行查询。

<br />

#### 将子查询转换为 semi-join

---

不进行物化操作直接把子查询转换为连接查询是否可行呢？

我们来一步一步的分析一下，下面两个查询是十分相似的：

```mysql
-- 查询 1
SELECT * FROM s1 
WHERE key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a');

-- 查询 2
SELECT s1.* FROM s1 
INNER JOIN s2 ON s1.key1 = s2.common_field 
WHERE s2.key3 = 'a';
```

我们这样理解查询 1 ，对于表 s1 的某条记录，若我们能在表 s2（准确的说是执行完`WHERE s2.key3 = 'a'`之后的结果集）中找到一条或多条记录，这些记录的 common_field 列的值等于表 s1 记录的 key1 列的值，那么该条表 s1 的记录就会被加入到最终的结果集（欸，这不和查询 2 很像吗）。

从上述理解入手，查询 1 和查询 2 不相似的地方在于，我们不能保证对于表 s1 的某条记录，表 s2 之中有多少条记录满足`s1.key1 = s2.common_field`这个条件。也就是说，我们不能保证 s1 表中的记录在查询 2 中不会被多次的加入到结果集中 。

如何解决查询 1 和查询 2 的差异，从而使得子查询可以被转换为连接查询呢？

MySQL 提出了一个新的概念，**半连接（英文：semi-join）**。半连接的意思就是，对于表 s1 的某条记录来说，我们只关心在表 s2 中是否存在与之匹配的记录，而不关心具体有多少条记录与之匹配，最终的结果集中只保留表 s1 的记录。

也就是说 MySQL 会将查询 1 看作下面的这句 SQL ，我们姑且称其为查询 3 吧。

```mysql
-- 查询 3
-- semi-join只是在 MySQL 内部采用的一种执行子查询的方式，不面向用户
SELECT s1.* FROM s1 
SEMI JOIN s2 ON s1.key1 = s2.common_field
WHERE key3 = 'a';
```

这只是抽象的东西，半连接的具体实现方式有以下几种：

* **Table Pullout（子查询中的表上拉）**。即当子查询的查询列表处只有主键或者唯一索引列时，可以直接把子查询中的表上拉到外层查询的 FROM 子句中，并把子查询中的搜索条件合并到外层查询的搜索条件中。例如：

  ```mysql
  -- 查询语句执行 Table Pullout
  -- 执行前（注意 key2 列是表 s2 的唯一二级索引列）
  SELECT * FROM s1 
  WHERE key2 IN (SELECT key2 FROM s2 WHERE key3 = 'a');
  
  -- 执行后
  SELECT s1.* FROM s1 
  INNER JOIN s2 ON s1.key2 = s2.key2 
  WHERE s2.key3 = 'a';
  ```

  为什么可以这样做？因为对于同一条表 s1 中的记录，不可能找到两条以上符合`s1.key2 = s2.key2`的记录。

* **DuplicateWeedout Execution Strategy （重复值消除）**。即建立临时表，消除重复值。例如：

  ```mysql
  -- 查询语句
  SELECT * FROM s1 
  WHERE key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a');
  
  -- 对查询语句建立临时表
  CREATE TABLE tmp (
      id PRIMARY KEY
  );
  ```

  在执行连接查询的过程中，每当某条表 s1 中的记录要加入结果集时，就把这条记录的 id 值加入到上述的临时表中。若添加成功，则表明这条表 s1 中的记录未曾加入到最终的结果集中，那么该记录就可以添加到最终的结果集中；若添加失败，则表明这条表 s1 中的记录加入到最终的结果集中过，直接丢弃这条记录即可。

  这种使用临时表消除 semi-join 结果集中的重复值的方式称之为 **DuplicateWeedout** 。

* **LooseScan Execution Strategy （松散索引扫描）**。当查询语句子查询对应的表可以使用到索引 A，并且子查询查询列表对应的列，就是索引 A 对应的列。此时将查询语句转换为半连接查询后，若以子查询对应的表为驱动表，扫描对应的索引 A，取出符合子查询查询条件的记录，去被驱动表中做匹配操作。这时，符合子查询查询条件的子查询记录，可能会有重复，但是重复的值一定都聚集在一块，并且符合子查询条件的子查询记录会按顺序排列。我们若不将驱动表符合子查询条件的每一条子查询记录都去被驱动表中查询，而是只取值相同的记录的第一条去做匹配操作，这种方式，就被称之为**松散索引扫描**。

* **Semi-Join Materialization Execution Strategy**。这个就是我们之前介绍过的先把外层查询 IN 子句中的不相关子查询进行物化，然后再把外层查询的表和物化表的连接的行为，这种行为本质上也算是一种 Semi-Join 。只不过由于物化表中没有重复的记录，所以可以直接将子查询转为连接查询。

* **FirstMatch Execution Strategy （首次匹配）**。即先取一条外层查询中的记录，然后到子查询的表中寻找符合匹配条件的记录，如果能找到一条，则将该外层查询的记录放入最终的结果集并且停止查找更多匹配的记录，如果找不到则把该外层查询的记录丢弃掉；然后再取下一条外层查询中的记录，重复上面这个过程。

以上就是半连接的执行策略，也是半连接的具体实现方式，我们可以使用这些策略来执行查询。

```mysql
-- 查询 4
SELECT * FROM s1 
WHERE key1 IN (SELECT common_field FROM s2 WHERE s1.key3 = s2.key3);

-- 将查询 4 转换成半连接查询，即查询 5 

-- 查询 5 
-- 我们可以对查询 5 使用 DuplicateWeedout、LooseScan、FirstMatch 等半连接执行策略来执行查询，当子查询的查询列表处只有主键或者唯一二级索引列时，还可以直接使用 Table Pullout 的策略来执行查询。
-- 注意，由于查询 4 的子查询是相关子查询，所以不能转换为物化表来执行查询。
SELECT s1.* FROM s1 
SEMI JOIN s2 ON s1.key1 = s2.common_field AND s1.key3 = s2.key3;
```

<br />

#### semi-join 的适用条件

---

包含 IN 子查询的查询语句转换为 semi-join，需要满足以下条件：

* 该子查询必须是和 IN 语句组成的布尔表达式，并且在外层查询的 WHERE 或者 ON 子句中出现。
* 外层查询可以有其他搜索条件，只不过和 IN 子查询的搜索条件必须使用 AND 关键字连接起来。
* 该子查询必须是一个单一的查询，不能是由若干查询由 UNION 连接起来的形式。
* 该子查询不能包含 GROUP BY 或 HAVING 语句 或 聚集函数。
* 还有一些条件比较少见，不再介绍。

<br />

#### 不适用于 semi-join 的情况

---

子查询无法转为 semi-join 的情况，典型的有以下几种：

* 外层查询的 WHERE 条件中有其他搜索条件与 IN 子查询组成的布尔表达式使用 OR 连接起来。

* 对子查询使用 NOT IN 关键字而不是 IN 关键字的情况。

* 在 SELECT 子句中对子查询使用 IN 关键字的情况。

  ```mysql
  -- 例子
  SELECT key1 IN (SELECT common_field FROM s2 WHERE key3 = 'a') FROM s1 ;
  ```

* 子查询中包含 GROUP BY 、 HAVING 关键字或 聚集函数的情况。

* 子查询中包含 UNION 关键字的情况。

但是对于不能转为 semi-join 查询的子查询，MySQL 仍能对其优化：

* **对于不相关的子查询**，可以尝试将其物化后再参与查询。

  ```mysql
  -- 例子
  SELECT * FROM s1 
  WHERE key1 NOT IN (SELECT common_field FROM s2 WHERE key3 = 'a')
  ```

  我们可以对上例中的子查询进行物化操作，再判断 key1 是否在物化表的结果集中。但是这里将子查询物化之后不能转为和外层查询表的连接（因为查询语句对子查询使用 NOT IN 关键字），只能是先扫描表 s1，然后对表 s1 的某条记录来说，判断该记录的 key1 值是否在物化表中。

* 无论子查询的相关与否，都可以把 IN 子查询**尝试**转为 EXISTS 子查询。

  其实对于任何一个 IN 子查询而言，都可以被转为 EXISTS 子查询。但是对于这个转换，是有特殊情况的，因为 NULL 值的存在，而 NULL 值作为操作数的表达式结果往往为 NULL 。这就会导致对子查询使用 IN 关键字还是 EXISTS 关键字，会有一定的区别。

  ```mysql
  -- 查询返回 NULL
  SELECT NULL IN (1, 2, 3);
  
  -- 查询返回 1（TRUE） / 0（FALSE）
  -- 实际测试下来应该是返回 1（TRUE） / NULL（FALSE）
  SELECT EXISTS (SELECT 1 FROM s1 WHERE NULL = 1);
  ```

  但是在 WHERE  或者 ON 子句中是不区分 NULL 和 FALSE 的。

  ```mysql
  -- 查询语句
  mysql> SELECT 1 FROM s1 WHERE NULL;
  -- 查询结果
  Empty set (0.00 sec)
  
  -- 查询语句
  mysql> SELECT 1 FROM s1 WHERE FALSE;
  -- 查询结果
  Empty set (0.00 sec)
  ```

  也就是说只要我们的 IN 子查询是放在 WHERE 或 ON 子句中，IN 子查询到 EXISTS 子查询的转换，就是没有问题的。

  为何要将 IN 子查询转换为 EXISTS 子查询呢？

  因为不转换可能用不到索引。例如：
  
  ```mysql
  -- 例子
  -- 该查询为相关子查询，且其子查询无法使用到索引，因为 common_field 字段没有索引
  SELECT * FROM s1
  WHERE key1 IN (
      SELECT key3 FROM s2
      WHERE s1.common_field = s2.common_field
  ) 
  OR key2 > 1000;
  
  -- 将例子转换为 EXISTS 子查询如下
  -- 该查询的子查询可以使用到表 s2 的 idx_key3 索引
  SELECT * FROM s1
  WHERE EXISTS (
      SELECT 1 FROM s2 
      WHERE s1.common_field = s2.common_field 
      AND s2.key3 = s1.key1
  ) 
  OR key2 > 1000;
  ```
  
  **注意：若 IN 子查询不满足转换为 semi-join 的条件，又不能转换为物化表 或 转换为物化表的成本太大。那么该子查询就会被转换为 EXISTS 子查询，且在 MySQL5.5 以及之前的版本没有引进 semi-join 和物化的方式优化子查询，在 MySQL5.5 以及之前的版本优化器都会把 IN 子查询转换为 EXISTS 子查询。**

<br />

#### 小结

---

IN 子查询的优化过程如下：

* 若 IN 子查询符合转换为 semi-join 的条件，查询优化器会优先把该 IN 子查询转换为 semi-join ，然后再考虑下面 5 种执行半连接的策略中，哪个成本最低：

  * Table pullout（子查询中的表上拉）
  * DuplicateWeedout（重复值消除）
  * LooseScan（松散索引扫描）
  * Materialization（物化）
  * FirstMatch（首次匹配）

  然后选择成本最低的那种执行策略来执行 IN 子查询。

* 若 IN 子查询不符合转换为 semi-join 的条件，那么查询优化器会从下面的两种策略中，找出一种成本更低的方式执行子查询：

  * 将 IN 子查询物化之后再执行查询。
  * 将 IN 子查询转换为 EXISTS 子查询。

<br />

## ANY / ALL 子查询优化

---

若 ANY / ALL 子查询是不相关子查询，那么他们能做如下转换：

|           原始表达式           |         转换后的表达式          |
| :----------------------------: | :-----------------------------: |
| < ANY (SELECT inner_expr ...)  | < (SELECT MAX(inner_expr) ...)  |
| \> ANY (SELECT inner_expr ...) | \> (SELECT MIN(inner_expr) ...) |
| < ALL (SELECT inner_expr ...)  | < (SELECT MIN(inner_expr) ...)  |
| \> ALL (SELECT inner_expr ...) | \> (SELECT MAX(inner_expr) ...) |

<br />

## [NOT] EXISTS 子查询的执行

---

若 [NOT] EXISTS 子查询是不相关子查询，则执行过程如下：

1. 先执行子查询，得出 [NOT] EXISTS 子查询的结果是 TRUE 还是 FALSE 。
2. 重写原先的查询语句。

举个例子：

```mysql
-- 例子
SELECT * FROM s1 
WHERE EXISTS (
    SELECT 1 FROM s2 
    WHERE key1 = 'a'
) 
OR key2 > 100;

-- 假设子查询结果为 TRUE，则上述查询语句可以写成下面这样
SELECT * FROM s1 
WHERE TRUE OR key2 > 100;

-- 再次化简得
SELECT * FROM s1 
```

若 [NOT] EXISTS 子查询是相关子查询，例如：

```mysql
SELECT * FROM s1 
WHERE EXISTS (
    SELECT 1 FROM s2 
    WHERE s1.common_field = s2.common_field
);

```

则执行过程如下：

1. 先从外层查询中获取一条记录，即先从 s1 表中获取一条记录。
2. 从上一步中获取的那条记录中找出子查询中涉及到的值，即从 s1 表中获取的那条记录中找出`s1.common_field`列的值，然后执行子查询。
3. 最后根据子查询的查询结果来检测外层查询 WHERE 子句的条件是否成立，如果成立，就把外层查询的那条记录加入到结果集，否则丢弃。
4. 再次执行第一步，获取第二条外层查询中的记录，依次类推。

这个过程若可以使用到索引，那查询速度会加快不少，例如：

```mysql
-- EXISTS 子查询中可以使用 idx_key1 索引来加快查询速度
SELECT * FROM s1 
WHERE EXISTS (
    SELECT 1 FROM s2 
    WHERE s2.key1 = s1.common_field
);
```

<br />

## 对于派生表的优化

---

我们首先来回忆以下派生表的概念，即放在 FROM 子句中的子查询，其结果集组成的表称之为**派生表**。例如：

```mysql
-- 下面的 t 表便是派生表
SELECT m, n FROM (
    SELECT m2 + 1 AS m, n2 AS n 
    FROM t2 
    WHERE m2 > 2
) AS t;
```

对含有派生表的查询，MySQL 提供了两种执行策略：

* 把派生表物化。即将派生表的结果集写到一个内部的临时表中，然后将这个物化表当作普通表参与查询。

  需要注意的是，这里对派生表进行物化时，MySQL 使用了一种称为**延迟物化**的策略，也就是在查询中真正使用到派生表时，才会去尝试物化派生表，而不是还没开始执行查询就把派生表物化。例如：

  ```mysql
  -- 该查询执行时首先会到表 s2 中找出满足条件 s2.key2 = 1 的记录
  -- 若表 s2 中没有满足条件的记录，则说明参与连接的表 s2 记录就是空的，所以整个查询的结果集就是空的，这时候也没有必要去物化查询中的派生表了。
  SELECT * FROM (
  	SELECT * FROM s1 
  	WHERE key1 = 'a'
  ) AS derived_s1 
  INNER JOIN s2 ON derived_s1.key1 = s2.key1
  WHERE s2.key2 = 1;
  ```

* 将派生表和外层查询合并。即将派生表与外层查询的表合并，然后将派生表中的搜索条件放到外层查询的搜索条件中，例如：

  ```mysql
  -- 例1
  -- 合并前
  SELECT * FROM (
      SELECT * FROM s1 
      WHERE key1 = 'a'
  ) AS derived_s1;
  
  -- 合并后
  SELECT * FROM s1 
  WHERE key1 = 'a';
  
  -- 例2
  -- 合并前
  SELECT * FROM (
  	SELECT * FROM s1 
  	WHERE key1 = 'a'
  ) AS derived_s1 
  INNER JOIN s2 ON derived_s1.key1 = s2.key1
  WHERE s2.key2 = 1;
  
  -- 合并后
  SELECT * FROM s1 
  INNER JOIN s2 ON s1.key1 = s2.key1
  WHERE s1.key1 = 'a' 
  AND s2.key2 = 1;
  ```

  成功的合并派生表和外层查询，消除了派生表，这意味着，我们没有必要再付出创建和访问临时表的成本。但不是所有带派生表的查询都能被成功的和外层查询合并，当派生表中有这些语句就不可以和外层查询合并：

  * 聚集函数
  * DISTINCT
  * GROUP BY
  * HAVING
  * LIMIT
  * UNION 或 UNION ALL
  * 派生表对应子查询的 SELECT 子句中含有另一个子查询
  * 还有些不常用的情况，不再细说

MySQL 在执行带有派生表的查询的时候，会优先尝试把派生表和外层查询合并来执行查询；若不能合并，才会把派生表物化来执行查询。

















