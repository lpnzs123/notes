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

如何排除，当然是使用 WHERE 关键字了。这一种在外连接查询中，指定的 WHERE 子句中包含被驱动表中的列不为 NULL 值的条件称之为**空值拒绝（Reject-NULL）**。

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
  -- 子查询结果集组成的表称之为派生表，下面的 t 表便是派生表
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
WHERE key1 = (SELECT common_field FROM s2 WHERE s1.key3 = s2.key3 LIMIT 1);
```

执行过程如下：

* 先从外层查询中获取一条记录，即先从 s1 表中获取一条记录。
* 从上一步中获取的那条记录中找出子查询中涉及到的值，即从 s1 表中获取的那条记录中找出`s1.key3`列的值，然后执行子查询。
* 最后根据子查询的查询结果来检测外层查询 WHERE 子句的条件是否成立，如果成立，就把外层查询的那条记录加入到结果集，否则丢弃。
* 再次执行第一步，获取第二条外层查询中的记录，依次类推。

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



























