# Mysql是怎样运行的：第二十一章笔记

---

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

上述的将 redo 日志写入redo 日志文件组的行为会产生一个问题，就是写满了最后一个 ib_logfilen 文件，重新从头开始写，就会覆盖掉 ib_logfile0 文件中的日志信息。这里 InnoDB 采用了 checkpoint 来解决这个问题。

总共的 redo 日志文件大小为：innodb_log_file_size × innodb_log_files_in_group 。

<br />

### redo 日志文件格式

---



















