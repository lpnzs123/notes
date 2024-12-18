# Mysql是怎样运行的：第十八章笔记

---

## 缓存的重要性

---

InnoDB 存储引擎在处理客户端的请求时，当需要访问某个页的数据时，就会把完整的页的数据从磁盘全部加载到内存中（即使只访问一个页的一条记录，那也需要先把整个页的数据加载到内存中）。在进行完读写访问之后并不着急把该页对应的内存空间释放掉，而是将其**缓存**起来，这样将来有请求再次访问该页面时，就可以省去磁盘 IO 的开销了。

<br />

## InnoDB 的 Buffer Pool

---

为了缓存磁盘中的页，MySQL 服务器在启动时就会向操作系统申请一片**连续**的内存，这片内存便名为 **Buffer Pool（缓冲池）**，默认大小为 128M。可以在启动服务器的时候配置`innodb_buffer_pool_size`值，它表示 Buffer Pool 的大小。

```mysql
[server]
# Buffer Pool 大小单位为字节（Byte），最小值为 5M，设置小于最小值的大小会自动设置为 5M。
# 268435456 / 1024 / 1024 = 256M
innodb_buffer_pool_size = 268435456
```

<br />

### Buffer Pool 的内部组成

---

Buffer Pool 中默认的缓存页大小和磁盘上默认的页大小均为 16KB。每个缓存页都有一些所谓的**控制信息**。控制信息中包含了该页所属的表空间编号、页号、缓存页在 Buffer Pool 中的地址、链表节点信息、一些锁信息以及 LSN 信息等。

我们将每个页对应的控制信息占用的一块内存称为一个**控制块**，每个缓存页的控制块大小相同，控制块和缓存页是一一对应的，它们均被存放到了 Buffer Pool 中。

Buffer Pool 对应的内存空间结构如下（按序号从左到右在内存空间排列）：

1. 控制块部分：存放着与缓存页一一对应的控制块们。
2. 碎片：在分配足够多的控制块和缓存页后，可能剩余的空间不够一对控制块和缓存页的大小，那么这点空间就不会被用到，成为了碎片。
3. 缓存页部分：存放着与控制块一一对应的缓存页们。

**注意：控制块部分的第一个控制块对应缓存页部分的第一个缓存页，以此类推。**

每个控制块约占用缓存页大小的 5%，在 MySQL 5.7.21 版本中，每个控制块占用的大小是 808 字节。设置的`innodb_buffer_pool_size`值并不包含控制块部分的大小，InnoDB 在为 Buffer Pool 向操作系统申请连续的内存空间时，这片连续的内存空间一般会比`innodb_buffer_pool_size`的值大 5% 左右。

<br />

### free 链表的管理

---

初启动 MySQL 服务器时，需要完成对 Buffer Pool 的初始化，初始化过程如下：

1. 向操作系统申请 Buffer Pool 的内存空间。
2. 将 Buffer Pool 的内存空间划分成若干对控制块和缓存页。

此时并没有真实的磁盘页被缓存到了 Buffer Pool 中，因为还没有用到真实的磁盘页。

那当我们需要用到真实的磁盘页时，它应该放到 Buffer Pool 中哪个缓存页的位置呢？或者说，我们如何区分 Buffer Pool 中哪些缓存页是空闲的？

MySQL 将所有空闲缓存页对应的一个个控制块作为一个个节点放到一个**双向链表**中，这个链表就是 **free 链表（又称空闲链表）**。假设 Buffer Pool 中可容纳的缓存页数量为 n，那么初始时，free 链表节点的数量就为 n。

free 链表有一个**基结点**，里面包含了 free 链表的头节点地址，尾节点地址，以及当前链表中节点的数量等信息。需要注意的是，这个基结点占用的内存空间并不包含在 `innodb_buffer_pool_size`值指定的 Buffer Pool 的大小中，而是在一块单独申请的内存空间中。

**注意：链表基结点占用的内存空间并不大，在 MySQL 5.7.21 版本中，每个链表基结点只占用 40 Bytes。后边介绍的许多不同链表，它们的基节点和 free 链表基节点的内存分配方式是一样的，都是单独申请的一块 40 Bytes 大小的内存空间，并不包含在为 Buffer Pool 申请的一大片连续内存空间之内。**

free 链表的每个节点（控制块）中都包含着 free 链表的 pre 和 next 指针。

这样，我们就知道如何缓存真实的磁盘页了，步骤如下：

1. 从 free 链表中取一个空闲的缓存页，并把该缓存页对应的控制块信息填上（即该页所在的表空间、页号之类的信息）。
2. 把该缓存页对应的 free 链表节点从链表中移除，表示该缓存页已经被使用了。

<br />

### 缓存页的哈希处理

---

当我们需要从 Buffer Pool 中使用我们已经缓存的磁盘页时，我们应该如何快速定位到对应的缓存页？遍历？不行，最快的当然是哈希表。

我们将 表空间号+页号 作为 key，缓存页作为 value 创建一个哈希表，在访问某个页的数据时，先从哈希表中根据 表空间号 + 页号 看看有没有对应的缓存页，有则直接使用即可，无则从 free 链表中选一个空闲的缓存页，然后把磁盘中对应的页加载到该缓存页的位置。

<br />

### flush 链表的管理

---

当我们修改了 Buffer Pool 中某个缓存页的数据，那么这个缓存页就和对应的磁盘页数据不一致了，这时，这个缓存页就被称为**脏页（英文：Dirty Page）**。

在每次修改缓存页后，我们并不会着急的将其修改立即同步到磁盘上，而是会在未来的某个时间点进行同步。并且，我们也不是一次性将所有的缓存页都与磁盘进行同步，而只是会同步那些修改过的缓存页。

但我们怎么知道哪些缓存页被修改过呢？我们需要一个和 free 链表原理一样的，用于存储脏页的链表，即**flush 链表**。

flush 链表与 free 链表的构造一致。对于修改过的缓存页，缓存页对应的控制块都会作为一个节点加入到 flush 链表中。

flush 链表也有一个**基结点**，其与 free 链表的基结点性质与结构一致。

<br />

### LRU 链表的管理

---

#### 缓存不够的窘境

---

当 Buffer Pool 的内存被用尽，但要缓存新的磁盘页时。我们需要移除 Buffer Pool 中旧的缓存页，然后再把新的页放入 Buffer Pool。

移除哪些旧的缓存页呢？不如说，保存哪些缓存页不被移除呢？

我们当然希望缓存命中率越高的、最近使用越频繁的缓存页留下，而最近很少使用的缓存页将被淘汰。

<br />

#### 简单的 LRU 链表

---

最近很少使用的缓存页，我们如何识别呢？

同样的，我们可以建立一个链表，这个链表按照**最近最少使用**的原则去淘汰缓存页，即 **LRU 链表（LRU 全称为 Least Recently Used）**。

这样，当我们需要访问某个页时，有以下两种情况：

* 该页没有被 Buffer Pool 缓存。在把该页从磁盘加载到 Buffer Pool 缓存页的过程中，就把该缓存页对应的控制块作为节点塞到链表的头部。
* 该页已被 Buffer Pool 缓存。则直接把该页对应的控制块移动到 LRU 链表的头部。

这样，我们就能保证 LRU 链表的尾部，就是最近最少使用的缓存页。

当 Buffer Pool 中的空闲缓存页使用完时，到 LRU 链表的尾部找些缓存页淘汰即可解决缓存不够的窘境。

**注意：LRU 链表与 free 链表的构造一致，且 LRU 链表也有一个**基结点**，其与 free 链表的基结点性质与结构一致。**

<br />

#### 划分区域的 LRU 链表

---

简单的 LRU 链表并不能完全解决缓存不够的窘境。尤其是遇到以下两种情况时：

* 情况一：InnoDB 存在**预读**（英文名：Read Ahead）行为，可能会导致”劣币驱逐良币“。

  所谓预读，即 InnoDB 认为执行当前的请求，可能在此之后会读取到某些页面，就预先把它们加载到 Buffer Pool 中。**根据触发方式的不同**，预读又分两种：

  * 线性预读

    存在系统变量`innodb_read_ahead_threshold`，若顺序访问某个区的页面数量超过了这个系统变量的值，就会触发依次**异步**读取，这次读取请求会将下一个区中的所有页全部缓存到 Buffer Pool 中，这就是线性预读。

    系统变量`innodb_read_ahead_threshold`的默认值为 56。我们可以在服务器启动时通过启动参数或者在服务器运行过程中直接调整该系统变量的值。不过注意，系统变量`innodb_read_ahead_threshold`是全局变量，需使用`SET GLOBAL`命令来进行修改。

    > InnoDB 实现异步读取，在 Windows 或者 Linux 平台上，可能是直接调用操作系统内核提供的 AIO 接口。在其它类 Unix 操作系统中，使用了一种模拟 AIO 接口的方式来实现异步读取，其实就是让别的线程去读取需要预读的页面。

   * 随机预读
  
     若 Buffer Pool 中已经缓存了某个区的 13 个连续的页面，不论这些页是否是顺序读取，都会触发一次异步读取本区中所有其他的页面到 Buffer Pool 中的请求，这就是随机预读。
     
     通过系统变量`innodb_random_read_ahead`，我们可以控制随机预读功能的开启与关闭，其值默认为 OFF。若我们想开启该功能，可以通过修改启动参数或者直接使用`SET GLOBAL`命令把该变量的值设置为 ON 。
  
  预读这一行为，在 Buffer Pool 容量不大且很多预读的页面都没有用到的时候，会导致处在 LRU 链表尾部的一些可能使用频率很高的缓存页很快的淘汰掉（因为预读的页都会放到 LRU 链表的头部，当然就会把原来在头部的挤到尾部去），即”劣币驱逐良币“，这样的结果就是会大大降低缓存命中率。
  
* 情况二：全表扫描若表中数据量庞大，会导致 Buffer Pool 整个”换血“，影响其他查询语句对 Buffer Pool 的使用。

  全表扫描意味着会访问到该表所在的所有页，假设这个表中的记录数量庞大，那么访问到的页也就会特别的多。这样，当需要访问到这些页时，就需要把它们统统加载到 Buffer Pool 中，整个 Buffer Pool 就被“换血”了！导致其他查询语句在执行时又得执行一次从磁盘加载到  Buffer Pool 的操作，大大降低了缓存命中率。

针对上述情况，InnoDB 将 LRU链表按一定比例分成两部分，分别是：

* 存储使用频率非常高的缓存页的一部分。这一部分链表也叫**热数据**，或称 **young 区域**。
* 存储使用频率不是很高的缓存页的一部分。这一部分链表也叫**冷数据**，或称 **old 区域**。

**注意：按比例将 LRU 链表分成两部分，并不是说某些节点就在 young 区域或 old 区域中固定不动了。而是随着程序的运行，某个节点所属的区域也可能发生变化。**

系统变量 innodb_old_blocks_pct 的值，确定了 old 区域在 LRU 链表中所占的比例。

```mysql
SHOW VARIABLES LIKE 'innodb_old_blocks_pct';

-- old 区域在 LRU 链表中所占的比例为 37%
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_old_blocks_pct | 37    |
+-----------------------+-------+

-- 我们可以修改配置文件来控制 old 区域在 LRU 链表中所占的比例
-- 这里我们修改 old 区域在 LRU 链表中所占的比例为 40%
[server]
innodb_old_blocks_pct = 40

-- 我们也可以在服务器运行期间，修改 innodb_old_blocks_pct 系统变量的值
-- 注意 innodb_old_blocks_pct 系统变量属于全局变量，一经修改将对所有客户端生效
-- 这里我们修改 old 区域在 LRU 链表中所占的比例为 40%
SET GLOBAL innodb_old_blocks_pct = 40;
```

下面，我们来解决预读和全表扫描分别造成的”劣币驱逐良币“和”换血“两个问题。

* 解决”劣币驱逐良币“问题。InnoDB 规定，**磁盘页初次被加载到 Buffer Pool 中的某个缓存页中时，该缓存页对应的控制块会被放到 old 区域的头部。**这样被预读到 Buffer Pool 却不进行后续访问的页面就会逐渐的从 old 区域离开，而不会影响到 young 区域中使用频率较高的缓存页。

* 解决”换血“问题。被加载到 Buffer Pool 的磁盘页，其对应缓存页对应的控制块，在第一次被访问时就会被加入到 young 区域头部（未被访问时在 old 区域中）。这样做的后果就是仍然会把 young 区域那些使用频率比较高的页面给顶下去。我们无法通过“限制第一次访问缓存页不将其控制块放入 young 区域，后续访问再将其控制块放入 young 区域”的行为来解决上述问题。因为一个页面中可能会包含多条记录，而每读取一条记录，就是访问一次页面，这样的话，即使“限制第一次访问缓存页不将其控制块放入 young 区域，后续访问再将其控制块放入 young 区域”，也大概率没有效果。

  针对上述问题，InnoDB 规定，**对某个处在 old 区域的缓存页进行第一次访问时，会在它对应的控制块中记录下对应访问时间。若后续的访问时间与第一次的访问时间在某个时间间隔内，那么该页面的控制块就不会被从 old 区域移动到 young 区域的头部，否则就会将它移动到 young 区域的头部。**

  这个时间间隔由系统变量`innodb_old_blocks_time`控制，默认值是 1000 ms。

  ```mysql
  SHOW VARIABLES LIKE 'innodb_old_blocks_time';
  
  -- innodb_old_blocks_time 默认值为 1000，单位为毫秒
  -- 我们可以在服务器启动或运行时设置 innodb_old_blocks_time 的值
  -- 若 innodb_old_blocks_time 的值为 0，表示每次访问一个缓存页时，就会把该缓存页对应的控制块移动到 young 区域的头部
  +------------------------+-------+
  | Variable_name          | Value |
  +------------------------+-------+
  | innodb_old_blocks_time | 1000  |
  +------------------------+-------+
  ```

上述的两个 InnoDB 的规定，极大程度**遏制**了”劣币驱逐良币“和”换血“这两个问题，因为用不到的预读页面以及全表扫描的页面都只会被放到 old 区域，而不影响 young 区域中的缓存页。

<br />

#### 更进一步优化 LRU 链表

---

“每访问一个缓存页时，就将其对应的控制块移动到 young 区域的头部”开销是否过高了？

有许多针对这一点的优化策略，我们简述一个：

**young 区域的头 1 / 4 的部分，其缓存页就算被再次访问，也不会将其移动到 young 区域的头部。**

这样，我们就降低了调整 LRU 链表的频率，适当的提升了性能。

更多的 LRU 链表的优化措施这里不再赘述。

> 我们在了解上述概念后需要重构一下随机预读的概念。那就是，若 Buffer Pool 中已经缓存了某个区的 13 个连续的页面，并且这 13 个页面都是非常热的页面（即缓存页对应的控制块在整个 young 区域的头 1 / 4 处），那么不论这些页是否是顺序读取，都会触发一次异步读取本区中所有其他的页面到 Buffer Pool 中的请求，这就是随机预读。

<br />

### 其他的一些链表

---

为了更好的管理 Buffer Pool 中的缓存页，除了上文中的措施，InnoDB 还提供了其他的链表或数据结构来对 Buffer Pool 中的缓存页进行管理，这里列出部分，如：

* unzip LRU 链表
* zip clean 链表
* zip free数组

具体详情不再赘述，请自行查阅其功能或源码。

<br />

### 刷新脏页到磁盘

---

后台有专门的线程，每隔一段时间负责把脏页刷新到磁盘（这样就不会影响用户线程处理正常的请求了），刷新方式有两种：

* 从 LRU 链表的 old 区域中刷新一部分页面到磁盘（更详细来说是从 LRU 链表的尾部开始扫描刷新，扫描的页面数可以通过系统变量 `innodb_lru_scan_depth`指定）。这种刷新方式被称之为 **BUF_FLUSH_LRU**。

* 从 flush 链表中刷新一部分页面到磁盘（刷新的速率取决于当时系统的繁忙程度）。这种刷新方式被称之为 **BUF_FLUSH_LIST**。

  * 当用户线程准备加载一个磁盘页到 Buffer Pool 中，而 Buffer Pool 中没有可用的缓存页时，这时就会看看 LRU 链表的尾部，**有没有可以释放掉的，未修改的页面**，有则释放，而后磁盘页加载到 Buffer Pool 中。无则会将 LRU 链表尾部的一个脏页同步刷新到磁盘，而后存放这个已刷新脏页的缓存页，会被用于加载新的磁盘页。这种刷新单个页面到磁盘中的刷新方式被称之为 **BUF_FLUSH_SINGLE_PAGE**。

  * 当系统过于繁忙时，也可能会出现用户线程批量的从 flush 链表中刷新脏页的情况。
  
  **注意：在处理用户请求的过程中去刷新脏页，会严重的降低处理处理用户请求的速度。**

<br />

### 多个 Buffer Pool 实例

---

在 Buffer Pool 特别大而且多线程并发访问特别高的情况下（多线程环境下，Buffer Pool 中的各种链表都需要加锁处理），单一的 Buffer Pool 可能会影响请求的处理速度，这时候就需要将其拆分成若干个小的 Buffer Pool，每个小的 Buffer Pool 都被称之为一个**实例**，实例间互相独立（独立的去申请内存空间，独立的管理各种链表等）。这样，在多线程并发访问时就不会相互影响了，提高了并发处理的能力。

Buffer Pool 实例的个数可以通过修改系统变量`innodb_buffer_pool_instances`的值（服务器启动时设置）来改变，例如：

```mysql
[server]
# Mysql 会创建两个 Buffer Pool 实例
# 每个实例实际占的内存空间 = innodb_buffer_pool_size（Buffer Pool 总大小） / innodb_buffer_pool_instances（Buffer Pool 实例个数）
innodb_buffer_pool_instances = 2
```

Buffer Pool 实例并不是越多越好，因为管理这些实例也需要性能开销。

**InnoDB 规定：innodb_buffer_pool_size 设置的 Buffer Pool 总共的大小不超过 1G 时，innodb_buffer_pool_instances 的值始终为 1（修改 Buffer Pool 实例个数的行为会无效）。**

因此，我们建议，在 Buffer Pool 的总大小大于或等于 1G 时，再设置多个 Buffer Pool 实例。 

<br />

### innodb_buffer_pool_chunk_size

---

在 MySQL 5.7.5 之前，Buffer Pool 的大小只能在服务器启动时通过配置`innodb_buffer_pool_size`启动参数来调整大小，在服务器运行过程中是不允许调整该值的。但在 MySQL 5.7.5 之后，支持了在服务器运行过程中调整 Buffer Pool 大小的功能。这引发了一个问题，那就是，每次调整 Buffer Pool 大小，都需要重新向操作系统申请一块连续的内存空间，而后将旧的 Buffer Pool 中的内容复制到新的内存空间中，这极其耗时。

所以，**MySQL 将每个 Buffer Pool 实例又细分切割为了许多个 chunk，一个 chunk 就代表了一片连续的内存空间。**也就是说，MySQL 不再一次性为某个 Buffer Pool 实例向操作系统申请一大片连续的内存空间，而是以 chunk 为单位向操作系统申请空间（将整体分割为部分，然后修改只对部分生效不涉及整体）。

现在一个 Buffer Pool 实例对应的内存空间结构如下：

1. chunk 部分，每个 chunk 结构如下（按序号从左到右在内存空间排列）：
   1. 控制块部分：存放着与缓存页一一对应的控制块们。
   2. 碎片：在分配足够多的控制块和缓存页后，可能剩余的空间不够一对控制块和缓存页的大小，那么这点空间就不会被用到，成为了碎片。
   3. 缓存页部分：存放着与控制块一一对应的缓存页们。
2. 各种链表（如 free 链表、flush 链表和 LRU 链表等）。

既然有了 chunk，那么在服务器运行期间调整 Buffer Pool 的大小时就是以 chunk 为单位进行增加或者删除内存空间了，而不是说对整个 Buffer Pool 的大小进行操作，之后进行缓存页的赋值。

chunk 的大小可以在启动操作 MySQL 服务器时，通过`innodb_buffer_pool_chunk_size`启动参数指定，其默认值为 134217728 Bytes（即 128 MB）。

**注意：`innodb_buffer_pool_chunk_size`的值只能在服务器启动时指定，在服务器运行过程中是不可以修改的。因为在服务器运行过程中修改`innodb_buffer_pool_chunk_size`的值意味着和调整 Buffer Pool 大小的值一样，我们需要重新向操作系统申请一块连续的内存空间，而后将旧的 chunk 中的内容复制到新的内存空间中，耗时的问题又出现了。本质上来说，又成了整体修改的行为，而不是部分修改。另外，因为`innodb_buffer_pool_chunk_size`的值并不包含缓存页对应控制块的内存空间大小，所以 InnoDB 向操作系统申请连续内存空间时，每个 chunk 的大小要比 `innodb_buffer_pool_chunk_size`的值大一些，约 5%。**

<br />

### 配置 Buffer Pool 时的注意事项

---

**为了保证每一个 Buffer Pool 实例中，包含的 chunk 数量相同，`innodb_buffer_pool_size`的值（Buffer Pool 的总大小）必须是`innodb_buffer_pool_chunk_size × innodb_buffer_pool_instances`乘积（每个 Buffer Pool 实例中含有一个 chunk，所有 chunk 大小的总和，即 Buffer Pool 至少含有 chunk 的总大小）的整数倍。**

根据上述规定，有以下几种情况：

* `innodb_buffer_pool_size`的值与`innodb_buffer_pool_chunk_size × innodb_buffer_pool_instances`的乘积相等或为后者的整数倍。

  例如：服务器启动时，指定`innodb_buffer_pool_instances`的值为 16，`innodb_buffer_pool_chunk_size`的值不指定（取其默认为 128M）。那么`innodb_buffer_pool_size`指定的值必须为 2G（16 * 128 M / 1024 M = 2147483648 Bytes）或者为 2G 的整数倍。

  **注意：`innodb_buffer_pool_size`值单位为字节。**

* `innodb_buffer_pool_size`的值大于`innodb_buffer_pool_chunk_size × innodb_buffer_pool_instances`的乘积并且不是后者的整数倍。

  例如：服务器启动时，指定`innodb_buffer_pool_instances`的值为 16，`innodb_buffer_pool_chunk_size`的值不指定（取其默认为 128M），指定`innodb_buffer_pool_size`的值为 9G，此时服务器会自动把`innodb_buffer_pool_size`的值调整为 10G（10737418240 Bytes）。

* `innodb_buffer_pool_chunk_size × innodb_buffer_pool_instances`的乘积大于`innodb_buffer_pool_size`的值。

  例如：服务器启动时，指定`innodb_buffer_pool_size`的值为 2G，指定`innodb_buffer_pool_instances`的值为 16，指定`innodb_buffer_pool_chunk_size`的值被设置为 256M，此时`innodb_buffer_pool_chunk_size`的值会被服务器改写为`innodb_buffer_pool_size / innodb_buffer_pool_instances`的值，即 2G / 16 = 128M = 134217728 Bytes。

<br />

### Buffer Pool 中存储的其他信息

---

Buffer Pool 的缓存页除了可以用来缓存磁盘上的页面，还可以用来存储锁信息、自适应哈希索引等信息。

<br />

#### 查看 Buffer Pool 的状态信息

---

可以通过`SHOW ENGINE INNODB STATUS`这个 SQL 语句来查看关于 InnoDB 存储引擎，在运行过程中的一些状态信息，状态信息中包括了 Buffer Pool 的一些信息，详解如下：

* Total memory allocated：代表 Buffer Pool 向操作系统申请的，连续的内存空间的大小。包括了全部控制块、缓存页和碎片的大小。

* Dictionary memory allocated：为数据字典信息分配的内存空间大小。

  **注意：数据字典信息的内存空间大小与 Buffer Pool 无关，数据字典信息的内存空间大小不包括在 Total memory allocated 中。**

* Buffer pool size：代表该 Buffer Pool 中可以容纳多少缓存页。

* Free buffers：代表当前 Buffer Pool 还有多少空闲缓存页，即 free 链表中还有多少个节点。

* Database pages：代表 LRU 链表中页的数量，包含 young 和 old 两个区域的节点数量。

* Old database pages：代表 LRU 链表 old 区域的节点数量。

* Modified db pages：代表脏页的数量，即 flush 链表中的节点数。

* Pending reads：正等待从磁盘上加载到 Buffer Pool 中的页面数量。磁盘页加载到 Buffer Pool 中时，Buffer Pool 首先会为其分配一个缓存页以及缓存页对应的控制块，然后把这个控制块添加到 LRU 链表 old 区域的头部。但是此时磁盘页还未被加载到 Buffer Pool 中，这时，Pending reads 的值便会加 1。 

* Pending writes LRU：即将从 LRU 链表中刷新到磁盘中的页面数量。

* Pending writes flush list：即将从 flush 链表中刷新到磁盘中的页面数量。

* Pending writes single page：即将以单个页面的形式刷新到磁盘中的页面数量。

* Pages made young：代表 LRU 链表中曾经从 old 区域移动到 young 区域头部的节点数量。这里需要留意的是，一个节点只有从 old 区域移动到 young 区域头部，才会将 Pages made young 的值加 1。若该节点本来就在 young 区域的后 3 / 4 处，那么下一次访问这个页面时，虽然也会将它移动到 young 区域头部，但是这个过程并不会使  Pages made young 的值加 1。

* Page made not young：系统变量`innodb_old_blocks_time`设置的值若大于 0，首次访问或后续访问某个处在 old 区域的节点时，由于不符合时间间隔的限制而不能将其移动到 young 区域头部，会导致 Page made not young 的值加 1。这里需要留意的是，在 LRU 链表中，一个处在 young 区域头 1 / 4 处的节点，没有因为缓存页被访问而被移动到 young 区域的头部，这不会使得 Page made not young 的值加 1。

  **注意：young 区域的头 1 / 4 的部分，其缓存页就算被再次访问，也不会将其移动到 young 区域的头部。**

* youngs/s：代表每秒从 old 区域移动到 young 区域头部的节点数量。

* non-youngs/s：代表每秒由于不满足时间限制而不能从 old 区域移动到 young 区域头部的节点数量。

* Pages read、created、written：分别代表读取、创建和写入了多少页。后边跟着读取、创建和写入的速率。

* Buffer pool hit rate：表示在过去某段时间，平均访问1000次页面，有多少次该页面已经被缓存到了 Buffer Pool 中。

* young-making rate：表示在过去某段时间，平均访问1000次页面，有多少次访问使页面移动到了 young 区域的头部。这里需要留意的是，这里会统计两部分数据，分别是：

  * 从 old 区域移动到 young 区域头部的次数。
  * 从 young 区域移动到 young 区域头部的次数。即位于 young 区域后 3 / 4 的节点，因为其缓存页被访问而导致其被移动到 young 区域头部的次数。

* not (young-making rate)：表示在过去某段时间，平均访问1000次页面，有多少次访问没有使页面移动到 young 区域的头部。这里需要留意的是，这里会统计两部分数据，分别是：

  * 设置了`innodb_old_blocks_time`系统变量而导致访问了 old 区域中的节点，但没有把它们移动到 young 区域的次数。
  * 因为节点在 young 区域的头 1 / 4 处而没有被移动到 young 区域头部的次数。

* LRU len：代表 LRU 链表中节点的数量。

* unzip_LRU：代表 unzip_LRU 链表中节点的数量。

* I/O sum：最近 50s 读取磁盘页的总数。

* I/O cur：现在正在读取的磁盘页数量。

* I/O unzip sum：最近50s解压的页面数量。

* I/O unzip cur：正在解压的页面数量。

<br />

​    





























