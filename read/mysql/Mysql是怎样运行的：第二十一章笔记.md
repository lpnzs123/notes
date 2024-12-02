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

在将 redo 日志写入日志文件组时，会从 ib_logfile0 开始写，写满后往 ib_logfile1 中写，以此类推。当写到最后一个 ib_logfilen 文件时，就重新从 ib_logfile0 开始写。

上述将 redo 日志写入redo 日志文件组的行为会产生一个问题，就是写满了最后一个 ib_logfilen 文件，若从头开始写，就会覆盖掉 ib_logfile0 文件中的日志信息。这里 InnoDB 采用了 checkpoint 来解决这个问题。

总共的 redo 日志文件大小为：innodb_log_file_size × innodb_log_files_in_group 。

<br />

### redo 日志文件格式

---

log buffer 本质上是一片**连续的内存空间**，它被划分成了若干个 512 字节大小的 block（用来存储 redo 日志的页称为 redo log block，简称 block）。

将 log buffer 中的 redo 日志刷新到磁盘的**本质**，就是把 block 的镜像写入日志文件中。因此，redo 日志文件其实也是由若干个 512 字节大小的 block 组成的。

redo 日志文件组的每个文件大小都一致，格式也一致。其中，格式由两部分组成：

* 前 2048 个字节（即前 4 个 block）用来存储一些管理信息。
* 第 2048 个字节往后的部分，用来存储 log buffer 中的 block 镜像。

我们知道，当 redo 日志文件组的最后一个 redo 日志文件写满时，新增的 redo 日志会写到第一个 redo 日志文件的中。但是从第一个 redo 日志文件的哪一个地方写起呢？由 redo 日志文件格式可知，每个 redo 日志文件都是从第 2048 个字节往后存储 redo 日志的，redo 日志也就是从这里写起了。

这里我将 redo 日志文件第 2048 个字节往后的部分称为 **redo 日志文件的后半部分**，由上可以总结得出，redo 日志文件组中的 redo 日志文件的后半部分，按顺序从开始到结尾（第一个到最后一个 redo 日志文件）可以组成一个**循环链表**。

普通的 block 格式无非就是 log block header、log block body 和 log block trialer 三个部分，但是对于 redo 日志文件的前 4 个 block，它们的格式就和普通的 block 格式不一样了。 redo 日志文件的前四个 block 名称与格式分别如下（block 按序号顺序，从左到右排列）：

1. log file header：该字段描述了 redo 日志文件的一些整体属性，有属性如下：
   * LOG_HEADER_FORMAT（地址从 0 到 4B，占用 4 字节）：该属性表示 redo 日志的版本，例如在 MySQL 5.7.21 版本中该值永远为 1。
   * LOG_HEADER_PAD1（地址从 4B 到 8B，占用 4 字节）：该属性用作字节填充，无意义。
   * LOG_HEADER_START_LSN（地址从 8B 到 16B，占用 8 字节）：该属性标记了本 redo 日志文件开始的 LSN 值（即文件偏移量为 2048 字节初对应的 LSN 值）。
   * LOG_HEADER_CREATOR（地址从 16B 到 48B，占用 32 字节）：该属性为一个字符串，标记了本 redo 日志文件的创建者。正常运行时该值为 MySQL 的版本号，例如 MySQL 5.7.21。使用`mysqlbackup`命令创建的 redo 日志文件，该值为 ibbackup 和创建时间。
   * （地址从 48B 到 508B，占用 460 字节，没有被使用到）
   * LOG_BLOCK_CHECKSUM（地址从 508B 到 512B，占用 4 字节）：该属性表示本 block 的校验值，所有 block 都有，无需关心。
2. checkpoint1：该字段记录了一些关于 checkpoint 的属性，checkpoint1 结构如下：
   * LOG_CHECKPOINT_NO（地址从 0 到 8B，占用 8 字节）：该属性为服务器做 checkpoint 的编号，每做一次 checkpoint，该属性值加 1。
   * LOG_CHECKPOINT_LSN（地址从 8B 到 16B，占用 8 字节）：该属性为服务器做 checkpoint 结束时对应的 LSN 值，系统崩溃恢复时将从该属性值开始。
   * LOG_CHECKPOINT_OFFSET（地址从 16B 到 24B，占用 8 字节）：该属性为 LOG_CHECKPOINT_LSN 属性对应的 LSN 值，在 redo 日志文件组中的偏移量。
   * LOG_CHECKPOINT_LOG_BUF_SIZE（地址从 24B 到 32B，占用 8 字节）：该属性为服务器在做 checkpoint 操作时，对应的 log buffer 的大小。
   * （地址从 32B 到 508B，占用 476 字节，没有被使用到）
   * LOG_BLOCK_CHECKSUM（地址从 508B 到 512B，占用 4 字节）：该属性表示本 block 的校验值，所有 block 都有，无需关心。
3. 第三个 block 未被使用，忽略。
4. checkpoint2：结构与 checkpoint1 这个 block 的结构一致。

<br />

### Log Sequeue Number

---

























