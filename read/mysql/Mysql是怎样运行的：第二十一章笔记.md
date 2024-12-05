# Mysql是怎样运行的：第二十一章笔记

---

## 前言

---

本章笔记衔接第二十章笔记内容，不建议直接阅读本章而跳过第二十章笔记。

<br />

## redo 日志文件

---

### redo 日志刷盘时机

---

redo 日志不可能总在 log buffer 中（内存中）呆着，在某些情况下，redo 日志会被刷新到磁盘：

* log buffer 空间不足时。

  log buffer 的大小是有限的（通过系统变量`innodb_log_buffer_size`指定）。InnoDB 认为，若当前写入的 redo 日志量占了 log buffer 大小的一半左右，就需要把这些日志刷新到磁盘上。

* 事务提交时。

  在事务提交时可以不把修改过的 Buffer Pool 中的页面刷新到磁盘，但是为了保证持久性，必须把修改这些页面对应的 redo 日志刷新到磁盘。

* 后台线程**持续**刷新到磁盘。

  后台存在一个线程，大约每秒都会刷新一次 log buffer 中的 redo 日志到磁盘。

* **正常关闭**服务器时。

* checkpoint 时。

* 等等...

<br />

### redo 日志文件组

---

MySQL 的数据目录（使用`SHOW VARIABLES LIKE 'datadir'`查看）下，默认有两个名为 ib_logfile0 和 ib_logfile1 的文件，log buffer 中的日志默认情况下就是刷新到这两个磁盘文件中。

若我们对 MySQL 的上述默认行为不满意，我们可以通过下述的几个启动参数来进行对应的调整：

* innodb_log_group_home_dir：该参数指定了 redo 日志文件所在的**目录**，默认值就是 MySQL 当前的数据目录。
* innodb_log_file_size：该参数指定了每个 redo 日志文件的**大小**，在 MySQL 5.7.21 版本中，该参数默认值为 48 MB。
* innodb_log_files_in_group：该参数指定了 redo 日志文件的**个数**，默认值是 2，最大值是 100。

由上我们可知，redo 日志文件不止一个，而是一组，即**日志文件组**。组内的文件**命名格式**为：ib_logfile[数字]，数字从 0 开始依次递增。

在将 redo 日志写入日志文件组时，会从 ib_logfile0 开始写，写满后往 ib_logfile1 中写，以此类推。当写满最后一个 ib_logfilen 文件时，就又从 ib_logfile0 开始写。

上述将 redo 日志写入redo 日志文件组的行为会产生一个问题，就是写满了最后一个 ib_logfilen 文件，若从头开始写，就会覆盖掉 ib_logfile0 文件中的日志信息。这里 InnoDB 采用了 checkpoint 来解决这个问题。

总共的 redo 日志文件大小为：innodb_log_file_size × innodb_log_files_in_group 。

<br />

### redo 日志文件格式

---

log buffer 本质上是一片**连续的内存空间**，它被划分成了若干个 512 字节大小的 block（用来存储 redo 日志的页称为 redo log block，简称 block）。

将 log buffer 中的 redo 日志刷新到磁盘的**本质**，就是把 block 的镜像写入日志文件中。因此，redo 日志文件其实也是由若干个 512 字节大小的 block 组成的。

redo 日志文件组的每个文件大小都一致，格式也一致。redo 日志文件格式由两部分组成：

* 前 2048 个字节（即前 4 个 block）用来存储一些管理信息。
* 第 2048 个字节往后的部分，用来存储 log buffer 中的 block 镜像。

我们知道，当 redo 日志文件组的最后一个 redo 日志文件写满时，新增的 redo 日志会写到第一个 redo 日志文件的中。但是从第一个 redo 日志文件的哪一个地方写起呢？由 redo 日志文件格式可知，每个 redo 日志文件都是从第 2048 个字节往后存储 redo 日志的，redo 日志也就是从这里写起了。

这里我将 redo 日志文件第 2048 个字节往后的部分称为 **redo 日志文件的后半部分**，由上可以总结得出，redo 日志文件组中的 redo 日志文件的后半部分，按顺序从开始到结尾（第一个到最后一个 redo 日志文件）可以组成一个**循环链表**。

普通的 block 格式无非就是 log block header、log block body 和 log block trialer 三个部分，但是对于 redo 日志文件的前 4 个 block，它们的格式就和普通的 block 格式不一样了。 redo 日志文件的前四个 block 名称与格式分别如下（block 按序号顺序，从左到右排列）：

1. log file header：该字段描述了 redo 日志文件的一些整体属性，有属性如下：
   * LOG_HEADER_FORMAT（地址从 0 到 4B，占用 4 字节）：该属性表示 redo 日志的版本，例如在 MySQL 5.7.21 版本中该值永远为 1。
   * LOG_HEADER_PAD1（地址从 4B 到 8B，占用 4 字节）：该属性用作字节填充，无意义。
   * LOG_HEADER_START_LSN（地址从 8B 到 16B，占用 8 字节）：该属性标记了本 redo 日志文件开始的 lsn 值（即文件偏移量为 2048 字节初对应的 lsn 值）。
   * LOG_HEADER_CREATOR（地址从 16B 到 48B，占用 32 字节）：该属性为一个字符串，标记了本 redo 日志文件的创建者。正常运行时该值为 MySQL 的版本号，例如 MySQL 5.7.21。使用`mysqlbackup`命令创建的 redo 日志文件，该值为 ibbackup 和创建时间。
   * （地址从 48B 到 508B，占用 460 字节，没有被使用到）
   * LOG_BLOCK_CHECKSUM（地址从 508B 到 512B，占用 4 字节）：该属性表示本 block 的校验值，所有 block 都有，无需关心。
2. checkpoint1：该字段记录了一些关于 checkpoint 的属性，checkpoint1 结构如下：
   * LOG_CHECKPOINT_NO（地址从 0 到 8B，占用 8 字节）：该属性为服务器做 checkpoint 的编号，每做一次 checkpoint，该属性值加 1。
   * LOG_CHECKPOINT_LSN（地址从 8B 到 16B，占用 8 字节）：该属性为服务器做 checkpoint 结束时对应的 lsn 值，系统崩溃恢复时将从该属性值开始。
   * LOG_CHECKPOINT_OFFSET（地址从 16B 到 24B，占用 8 字节）：该属性为 LOG_CHECKPOINT_LSN 属性对应的 lsn 值，在 redo 日志文件组中的偏移量。
   * LOG_CHECKPOINT_LOG_BUF_SIZE（地址从 24B 到 32B，占用 8 字节）：该属性为服务器在做 checkpoint 操作时，对应的 log buffer 的大小。
   * （地址从 32B 到 508B，占用 476 字节，没有被使用到）
   * LOG_BLOCK_CHECKSUM（地址从 508B 到 512B，占用 4 字节）：该属性表示本 block 的校验值，所有 block 都有，无需关心。
3. 第三个 block 未被使用，忽略。
4. checkpoint2：结构与 checkpoint1 这个 block 的结构一致。

<br />

### Log Sequeue Number

---

存在一个全局变量`Log Sequeue Number`（中文为**日志序列号**，**简称 lsn**，专为 InnoDB 设计）用来记录当前系统已经写入的 redo 日志量，该全局变量的初始值为 8704，即一条 redo 日志也未写入时，lsn 的值为 8704。

全局变量`Log Sequeue Number`自系统开始运行便不断因为页面的修改（不断的页面修改意味着不断的 redo 日志的生成）而递增。在计算 **lsn 的增长量**时，我们需要把普通 block 格式的 log block header 和 log block trailer 大小也算进去。举个例子（注意结合普通的 block 结构理解）：

1. 系统第一次启动后，初始化 log buffer 时，全局变量 buf_free（该变量指明了后续写入 log buffer 的 redo 日志，应该写入到 log buffer 的哪一个位置）会指向 log buffer 的第一个 block 的偏移量为 12 字节的地方。我们知道，log block header 部分的大小就是 12 字节，因此 lsn 的值在此时为：8704 + 12 = 8716。
2. 执行 mtr_1（MySQL 把对底层页面中的一次原子访问过程称之为一个 mtr），假设其产生的一组 redo 日志大小为 200 字节，**没有超过**待插入的 block 的剩余空闲空间的大小，此时 lsn 的值为：8716 + 200 = 8916。
3. 执行 mtr_2，假设其产生的一组 redo 日志大小为 1000 字节，**超过了**待插入的 block 的剩余空闲空间的大小，必须再分配两个 block 才能装下，那么此时 lsn 的值为：8916 + 1000（mtr_2 产生的一组 redo 大小）+ 2 * 12（2 个 log block header 的大小）+ 2 * 4（2 个 log block trailer 的大小）= 9948。

综上所述，**每一组由 mtr 产生的 redo 日志，都有一个唯一的 lsn 值与其对应，lsn 值越小，说明 redo 日志组产生的越早**。

<br />

### flushed_to_disk_lsn

---

存在一个全局变量`buf_next_to_write`（专为 InnoDB 设计）用来标记当前的 log buffer 中，有哪些日志已经被刷新到磁盘中了。

全局变量 lsn 表示的是当前系统中写入的 redo 日志量，这其中包括了已经写入到 log buffer 中，但还没被刷新到磁盘的 redo 日志。为了记录刷新到磁盘中的 redo 日志量，InnoDB 又提出了一个新的全局变量`flushed_to_disk_lsn`。

此时，我们有了以下几个全局变量：

* buf_next_to_write（标明位置）：用来标记当前的 log buffer 中，有哪些日志已经被刷新到磁盘中了。
* flushed_to_disk_lsn（表示大小）：用来记录刷新到磁盘中的 redo 日志量。
* buf_free（标明位置）：指明了后续写入 log buffer 的 redo 日志，应该写入到 log buffer 的哪一个位置。
* Log Sequeue Number（表示大小）：用来记录当前系统已经写入的 redo 日志量

系统第一次启动时，全局变量`flushed_to_disk_lsn`和`Log Sequeue Number`的大小是相同的，都是 8704。但随着 redo 日志不断的写入 log buffer 中，两个全局变量的值会不断的拉开差距（明显的，新的 redo 日志刚被写入到 log buffer 中时，`Log Sequeue Number`的值会增长，`flushed_to_disk_lsn`的值会不变。随着不断有 log buffer 中的日志被刷新到磁盘上，`flushed_to_disk_lsn`的值才会跟着增长）。当两个全局变量的值再次相同时，就代表 log buffer 中所有的 redo 日志都已经刷新到磁盘中了。

**注意：应用程序（这里指 MySQL）向磁盘文件写入，其实是先写到操作系统的缓冲区中。若某个写入操作要等到操作系统确认已经写入磁盘了才会返回，那么此时需要调用一下操作系统提供的 fsync 函数。只有当系统执行了 fsync 函数了，全局变量`flushed_to_disk_lsn`的值才会跟着增长。当仅仅把 log buffer 中的 redo 日志写入到操作系统缓冲区中，却没有显式的将 redo 日志刷新到磁盘中时，只有另一个全局变量`write_lsn`会跟着增长。上面的叙述中，我们混淆了全局变量`flushed_to_disk_lsn`和全局变量`write_lsn`的概念，现在将其区分开来。**

<br />

### lsn 值和 redo 日志文件偏移量的对应关系

---

由每个 mtr 向磁盘中写入多少字节的日志，lsn 的值就增长多少，我们很容易得出 lsn 值在 redo 日志文件组中的偏移量。

例如，初始时 lsn 值为 8704（lsn 的初始值），对应文件偏移量是 2048（从 redo 日志文件的后半部分开始写）。经过两个 mtr 的执行后，文件偏移量增长到了 3292，那么此时 lsn 的值为：8704 +（3292 - 2048） = 9948。

<br />

### flush 链表中的 lsn 

---

一个 mtr 的执行，不仅可能会产生一组不可分割的 redo 日志，在 mtr 结束时，还会把这一组 redo 日志写入到 log buffer 中。但是 mtr 结束时要做的事情不止于此，mtr 结束时**还会把在 mtr 执行过程中可能修改过的页面对应的控制块，加入到 Buffer Pool 的 flush 链表中**。

第一次修改某个缓存在 Buffer Pool 中的页面时，会把这个页面对应的控制块插入到 flush 链表的**头部**，之后再次修改该页面时，由于其对应的控制块已经在 flush 链表中，所以就不再进行页面对应的控制块的二次插入了。即 **flush 链表中的脏页，是按照页面的第一次修改时间，从大到小进行排序的**。在控制块插入到 flush 链表的这个过程中，会在对应的控制块中记录两个关于页面何时修改的属性：

* oldest_modification：页面被加载到 Buffer Pool 中后，进行第一次修改时，会将修改该页面的 mtr 开始时对应的 lsn 值，写入到 oldest_modification 属性中。
* newest_modification：页面被加载到 Buffer Pool 中后，每一次修改时，都会将修改该页面的 mtr 结束时对应的 lsn 值，写入到 newest_modification 属性中。即该属性表示页面最近一次修改后，对应的系统 lsn 值。

举个例子：

1. 系统第一次启动后，初始化 log buffer 时，全局变量 buf_free（该变量指明了后续写入 log buffer 的 redo 日志，应该写入到 log buffer 的哪一个位置）会指向 log buffer 的，第一个 block 的，偏移量为 12 字节的地方。我们知道，log block header 部分的大小就是 12 字节，因此 lsn 的值在此时为：8704 + 12 = 8716。

2. 执行 mtr_1（**在 mtr_1 的执行过程中修改了页 a**），假设其产生的一组 redo 日志大小为 200 字节，没有超过待插入的 block 的剩余空闲空间的大小，此时 lsn 的值为：8716 + 200 = 8916。

   mtr_1 执行结束之时，会将页 a 对应的控制块，加入到 flush 链表的头部。并且将 mtr_1 刚开始执行时对应的 lsn 值（为 8716）写入到页 a 对应控制块的 oldest_modification 属性中，将 mtr_1 执行结束时对应的 lsn 值（为 8916）写入到页 a 对应控制块的 newest_modification 属性中。

3. 执行 mtr_2（**在 mtr_2 的执行过程中修改了页 b 和页 c，先修改页 c**），假设其产生的一组 redo 日志大小为 1000 字节，超过了待插入的 block 的剩余空闲空间的大小，必须再分配两个 block 才能装下，那么此时 lsn 的值为：8916 + 1000（mtr_2 产生的一组 redo 大小）+ 2 * 12（2 个 log block header 的大小）+ 2 * 4（2 个 log block trailer 的大小）= 9948。

   mtr_2 执行结束之时，会将页 b 和页 c 对应的两个控制块都加入到 flush 链表的头部，并且将 mtr_2 刚开始执行时对应的 lsn 值（为 8916）写入到页 b 和页 c 对应的两个控制块的 oldest_modification 属性中，将 mtr_2 执行结束时对应的 lsn 值（为 9948）写入到页 b 和页 c 对应的两个控制块的 newest_modification 属性中。

4. 执行 mtr_3（**在 mtr_3 的执行过程中修改了页 b 和页 d，先修改页 b**），假设其产生的一组 redo 日志大小为 52 字节，没有超过待插入的 block 的剩余空闲空间的大小，此时 lsn 的值为：9948 + 52 = 10000。

   mtr_3 执行结束之时，会将页 d 对应的控制块加入到 flush 链表的头部，并且将 mtr_3 刚开始执行时对应的 lsn 值（为 9948）写入到页 d 对应控制块的 oldest_modification 属性中，将 mtr_3 执行结束时对应的 lsn 值（为 10000）写入到页 d 对应控制块的 newest_modification 属性中。

   但是对于页 b，在mtr_3 执行结束之时，仅需要更新页 b 对应控制块的 newest_modification 属性值为 10000 即可。

综上所述，**flush 链表中的脏页，会按照修改发生的时间顺序进行排序，即按照 oldest_modification 属性代表的 lsn 值进行排序。被多次更新的页面，其对应的控制块不会重复的插入到 flush 链表中，但是会更新页面对应控制块的 newest_modification 属性值**。

<br />

### checkpoint

---

我们知道，在将 redo 日志写入日志文件组时（即写入磁盘时），会从 ib_logfile0 文件开始写，写满后往 ib_logfile1 文件中写，以此类推。当写满最后一个 ib_logfilen 文件时，就又从 ib_logfile0 文件开始写。

这样循环写入 redo 日志造成的问题就是会覆盖曾经写下的 redo 日志，但是，曾经的 redo 日志就不可以被覆盖了吗？当然是可以的。不过也不能随便覆盖，得有个判断标准，这个标准就是**被覆盖的 redo 日志对应的脏页，是否已经刷新到了磁盘里**。至于标准产生原因，如果对应的脏页已经刷新到了磁盘，还留着对应 redo 日志有什么用呢？系统崩溃重启也用不到这些日志呀。

也就是说，对于一个 mtr 修改的页 a（明显的，页 a 是脏页），若页 a 仍在 Buffer Pool 中未被刷新到磁盘，那 mtr 生成的一组 redo 日志在磁盘上的空间，是不可以被覆盖的。若页 a 被刷新到了磁盘，那么页 a 对应的控制块就会从 flush 链表中移除，mtr 生成的一组 redo 日志在磁盘上的空间，就可以被覆盖掉。InnoDB 存在一个**全局变量 checkpoint_lsn**，用来表示系统中可以被覆盖的 redo 日志总量，这个全局变量的初始值也是 8704。

现在假设一个 mtr 修改的页 a 被刷新到了磁盘，那么 mtr 生成的 redo 日志就可以被覆盖了。此时，我们可以进行一次增加全局变量 checkpoint_lsn 值的操作，这个过程就是**做一次 checkpoint**。

更详细来说，做一次 checkpoint 可以分成两步：

1. 计算一下当前系统中可以被覆盖的 redo 日志对应的 lsn 值最大是多少。

   也就是说，我们只要找到 Buffer Pool 的 flush 链表尾节点对应的 oldest_modification 的值，就知道了全局变量 checkpoint_lsn 的值，把 oldest_modification 的值赋给全局变量 checkpoint_lsn 就好了。

   因为凡是在系统 lsn 值小于 flush 链表尾节点 oldest_modification 值时产生的 redo 日志，都是可以被覆盖掉的（**即小于 checkpoint_lsn 值的  lsn 值对应的 redo 日志，都可以被覆盖掉，这里可以看出 checkpoint_lsn 本质上是一个 lsn 值**）。flush 链表尾节点代表当前系统中最早被修改，但还没有被刷新到磁盘的脏页。

2. 将 checkpoint_lsn、checkpoint_lsn 对应的 redo 日志文件组偏移量 checkpoint_offset 和此次 checkpoint 的编号（**这三个信息我简称为 ccc 信息**），写到 redo 日志文件的管理信息中（redo 日志文件的前 2048 个字节对应的 4 个 block 中的 checkpoint1 或 checkpoint2 中）。

   InnoDB 维护了一个**变量 checkpoint_no**，表示目前系统做了多少次 checkpoint。系统每做一次 checkpoint，变量 checkpoint_no 的值就加 1。

   我们知道，计算一个 lsn 值对应的 redo 日志文件组的偏移量是很容易的，所以可以计算得到 checkpoint_lsn 在 redo 日志文件组中对应的偏移量 checkpoint_offset。然后，我们就可以把 ccc 信息写到 redo 日志文件的管理信息中了。

   但是一组 redo 日志中有许多的 redo 日志文件，我们组内的每个 redo 日志文件的管理信息都要写入 ccc 信息吗？当然不是，ccc 的信息只会被写到 redo 日志文件组的第一个 redo 日志文件的管理信息中。问题又来了，写到管理信息的哪个 block 中呢？InnoDB 规定，当变量 checkpoint_no 的值是偶数时，就将 ccc 信息写到 checkpoint1 中，当变量 checkpoint_no 的值是奇数时，就将 ccc 信息写到 checkpoint2 中。


<br />

### 批量从 flush 链表中刷出脏页

---

如果当前系统修改页面的操作十分频繁，会导致写日志的操作也十分频繁，lsn 的值就会增长过快。这样呢，后台的刷脏操作（后台线程对 Buffer Pool 中的 LRU 链表和 flush 链表进行刷新脏页的操作）就可能不能及时的将脏页刷出，系统就无法及时的做 checkpoint。

这时，可能就需要用户线程同步的从 flush 链表中（平常的刷脏操作都是后台线程默默在做），把那些最早修改过的脏页刷新到磁盘了，在刷脏过后，这些脏页对应的 redo 日志就可以被覆盖了，就可以做 checkpoint 了。

<br />

   ### 查看系统中的各个 lsn 值

---

我们可以使用`SHOW ENGINE INNODB STATUS`这样的 SQL 语句查看当前 InnoDB 存储引擎中各个 lsn 值的情况。

```mysql
-- SQL 语句
SHOW ENGINE INNODB STATUS;

-- 当前 InnoDB 存储引擎中各个 lsn 值的情况
...
-- 系统中的 lsn 值，也就是当前系统已经写入的 redo 日志量，包括写入 log buffer 中的日志
Log sequence number 124476971
-- flushed_to_disk_lsn 的值，即刷新到磁盘中的 redo 日志量
Log flushed up to   124099769
-- Buffer Pool 的 flush 链表尾节点对应的 oldest_modification 属性值
Pages flushed up to 124052503
-- 当前系统的 checkpoint_lsn 值
Last checkpoint at  124052494
...
```

<br />

###  innodb_flush_log_at_trx_commit 的用法

---

































