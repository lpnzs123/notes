# PSQL的类型转换问题

## SQL举例

### 问题

```postgresql
UPDATE product
SET product_name = NULL
```

在PostgreSQL数据库中执行，是完全没有问题的，但是如果写成下面这样。

```postgresql
UPDATE product
SET product_name = 
	CASE WHEN pid = 1
		THEN NULL
	END
```

即会发生错误，并提示**你需要重写或转换表达式**。

<br/>

### 解决方案

若"product_name"字段的数据类型为varchar，我们将上述错误的SQL进行类型转换，便可以解决这个错误。

```postgresql
UPDATE product
SET product_name = 
	CASE WHEN pid = 1 
		THEN NULL::varchar
	END
```
（注意：此处case-when-then-end的条件（pid = 1）并无作用。也就是说，在此SQL语句更新product表时，会将product表中所有记录的"product_name"字段设置为NULL。）

<br/>

或者说，我们给case-when-then-end加上else，也可以解决这个问题。

```postgresql
UPDATE product
SET product_name = 
	CASE WHEN pid = 1 
		THEN NULL
		ELSE product_name
	END
```
（注意：此处case-when-then-end的条件（pid = 1）有作用。也就是说，在此SQL语句更新product表时，会将product表中pid为1记录的"product_name"字段设置为NULL，其余记录的product_name 字段保持不变。）


<br/>

但是注意，ELSE后面仍然不可以写NULL，即如下写法是错误的。

```postgresql
UPDATE product
SET product_name = 
	CASE WHEN pid = 1 
		THEN NULL
		ELSE NULL
	END
```

<br/>

此时进行类型转换，可以解决这个问题，即为如下SQL。

```postgresql
UPDATE product
SET product_name = 
	CASE WHEN pid = 2 
		THEN NULL::varchar
		ELSE NULL::varchar
	END
```

<br/>

### 总结

我将这个问题称为 case-when-then-null-end（CNE） 问题。