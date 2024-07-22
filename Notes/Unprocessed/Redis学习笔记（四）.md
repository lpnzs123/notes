<h2><span style="font-family: 黑体, 'Heiti SC'; font-size: 18pt;"><strong>一、 Redis持久化（默认使用RDB）</strong></span></h2>
<h3><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;"><strong>1.简要概述（概念理解）</strong></span></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）RDB（Redis Database）：即进行内存快照保存Redis整体数据，从而进行数据恢复。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）AOF（Append Only File）：即保存所有Redis执行过的命令，从而进行Redis数据恢复。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：Redis在内存中运行，即Redis运行时数据都在内存中。</span></p>
<p>&nbsp;</p>
<h3><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;"><strong>2.RDB进步实操</strong></span></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）RDB：以指定时间间隔执行数据集的时间点快照（概念），即在某一时刻把Redis数据和状态以文件的形式写到磁盘上面。这样的行为会产生一个名为 dump.rdb 的快照文件（二进制文件）。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）RDB操作分为自动触发和手动触发，以下介绍自动触发</span></p>
<p><img src="https://img2023.cnblogs.com/blog/3171133/202308/3171133-20230808173257318-745377050.png" alt="" width="398" height="295" /></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">图中的快照配置文件（即Redis配置文件SNAPSHOTTIN块）编辑命令方法，其命令行解析如下：</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">save为关键字，seconds单位为秒，changes即Redis数据库修改频次。总的来说，命令行的意义为给定的时间中，Redis数据库修改频次达到了我们给定的Redis数据库修改频次，则进行一次RDB操作。或者Redis数据库修改频次达到了我们给定的Redis数据库修改频次，也会进行一次RDB操作。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：执行flushall或flushdb或shutdown命令也会产生dump.rdb 快照文件，但是里面是空的，无意义。那么当我们错误的执行flushall或flushdb命令后想要恢复数据，就必须有以前的dump.rdb 快照文件，而每次执行RDB操作后，dump.rdb 快照文件都会更新到最新覆盖之前的dump.rdb 快照文件。所以，我们应该保证RDB快照文件存放的地方（/myredis/dumpfiles）和我们备份Redis数据库数据的地方要分开（即进行备份迁移）。建议是不将备份文件和生产Redis服务器放在同一台机器上，以防止生产机物理损坏后备份文件也跟着丢失。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">然后我们修改dump.rdb文件的保存路径</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;"><img src="https://img2023.cnblogs.com/blog/3171133/202308/3171133-20230808173257415-451935097.png" alt="" width="431" height="392" /></span></p>
<p align="left">&nbsp;</p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">将&rdquo;dir ./&rdquo;改为&rdquo;dir /myredis/dumpfiles&rdquo;。注意，dumpfiles文件夹必须存在，不可为空。在redis命令窗口中，我们可以通过config get dir查询我们的dump.rdb文件的保存路径是否修改成功。同样的，可以使用config get 来获取我们配置文件中其他的配置信息。例如，config get port获取我们Redis启动的端口号。还可以预见的是，config set可以对配置文件中的配置信息进行设定，例如config set stop-writes-on-bgsave-error no，</span><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">意思为后台持久化出错的时候不停止写入（数据一致性低）。这里注意，bgsave是从Redis主进程分出一个子进程进行持久化操作。上述命令行意思也为，当子进程bgsave出错，不对主进程有影响。此命令行的yes or no也决定了在RDB操作失败时，Redis数据库（主进程）是否继续接受新的写请求，此命令行默认yes。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">建议将默认的RDB生成的快照文件修改名字，以面对后面多台Redis同时运行的情况。修改的名字为：dump+端口号</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;"><img src="https://img2023.cnblogs.com/blog/3171133/202308/3171133-20230808173257431-968203163.png" alt="" width="471" height="432" /></span></p>
<p align="left">&nbsp;</p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）恢复数据则将我们RDB自动生成的dump.rdb文件放入我们指定的RDB的dump文件存放地址（/myredis/dumpfiles）再重启Redis服务即可。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">4）RDB手动触发（涉及到两个命令，save和bgsave。在生产环境中，只允许用bgsave不可以用save，RDB默认也使用bgsave）。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">save命令在主程序中执行时，会阻塞当前的redis服务器，直到持久化工作完成。在执行save&rsquo;命令期间，Redis不可以处理其他命令。而bgsave不会阻塞当前的redis服务器，而是在后台异步进行快照操作，该触发方式会fork一个子进程由子进程复制持久化过程。因此，我们在线上时，只用bgsave不用，save。因为后者一旦使用，线上的Redis就只能被阻塞去全力备份，其缓存功能全面失效，会导致严重的生产问题。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">5）lastsave命令可以获取最后一次成功执行快照的时间（返回一个unix时间戳）。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">6）&rdquo;date -d @unix时间戳&rdquo;即将时间戳转换成我们看的懂的年月日具体时间，在Redis命令窗口中使用。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">7）RDB在内存中的加载速度要比AOF快很多，并且在恢复大的数据集的时候，RDB的方式会更快。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">8）因为RDB是一段时间间隔内做一次备份，因此在上次备份和下次备份时间中Redis宕机了，则会丢失宕机到上次备份时间的时间段之中的数据。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">9）fork一个子进程进行持久化的过程可能会因为数据集的庞大导致严重影响服务器的性能。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">10）若dump文件有破损导致无法使用，可以尝试使用&rdquo;redis-check-rdb dump文件路径&rdquo;来进行修复，修复不了就报错。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">11）禁止使用RDB方式进行持久化。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第一种方式：redis-cli config set save &ldquo;&rdquo;，执行此段命令即可。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第二种方式：进入Redis配置文件中，将save关键字后改为 &ldquo;&rdquo; 即可。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：禁用RDB的情况下我们仍然可以使用save和bgsave命令生成rdb文件。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">12）在Redis配置文件SNAPSHOTTIN块中，我们可以通过如下设置来优化RDB持久化功能。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">rdbcompresion yes 即对应存储在磁盘中的快照设置是否进行压缩存储，若不想消耗CPU来进行压缩的话，可以设置关闭此功能。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">rdbchecksum yes 即启用RDB文件的合法性校验。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">rdb-del-sync-files no 即在主从复制实例的时候会产生RDB文件，若没有指定持久化则是否删除复制中的RDB文件（no即禁用）。</span></p>
<p align="left">&nbsp;</p>
<h3 align="left"><strong><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;">3.AOF进步实操（一）</span></strong></h3>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）AOF即以日志的形式来记录每一个写操作（读操作不记录），该日志文件只允许追加文件内容不允许改写文件内容。当Redis重启的时候会根据这个日志文件的内容将写指令从前到后执行一次以完成数据恢复的工作。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）默认情况下Redis并未开启AOF的功能，需要在Redis配置文件中开启，即在配置文件中将 appendonly no改为appendonly yes。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）AOF保存的文件名为 appendonly.aof文件，从客户端等将写命令发给服务端Redis，并不会立马将其写入AOF文件中，存储相应的命令到AOF缓存区中。当缓存区的命令达到一定量级的时候再将其写入到AOF文件中。这样，我们就避免了频繁的磁盘IO操作。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">4）AOF文件有一种自我优化机制，当AOF缓冲区写了越来越多的命令到AOF文件中时，AOF文件便会使用这种自我优化机制，即根据命令的规则进行命令的合并（也称AOF重写），以达到AOF文件压缩的目的。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">5）写命令从AOF缓冲区到AOF文件中，有三种写回策略（即将命令从AOF缓冲区写到AOF文件中的策略）。分别为，always（永远）、everysec（每隔一秒，默认）、no（无策略），可以在配置文件中配置写回策略，其配置文件项为appendfsync。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">解析</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">always：同步写回，即每个命令执行完成后立刻同步地将日志写回磁盘。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">everysec：每秒写回，即每个命令执行完，只是先把日志写到AOF内存缓冲区，每隔一秒把缓冲区中的内容写入磁盘。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">no：由操作系统控制写回，即每个命令执行完，只是先将日志写到AOF内存缓冲区，由操作系统决定何时将缓冲区的内容写回磁盘。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">6）AOF文件保存路径Redis6以前和Redis7不同，以下介绍Redis7</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">AOF文件保存路径是Redis配置项dir路径下的名为appendonlydir 的文件夹中（appendonlydir文件夹Redis会自己建立），这个appendonlydir文件夹名可以通过Redis配置文件中的appenddirname配置项进行修改。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：Redis以前6的AOF文件保存位置和RDB文件的保存位置一致，都是通过Redis配置的文件的dir配置项进行配置。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">7）AOF文件保存名称Redis6以前和Redis7，Redis6以前就是单纯的名为appendonly.aof，Redis7将单个AOF文件拆成了多个，将AOF文件分成了三个类型。分别是，BASE（基础AOF类型，一般由子进程通过重写产生，该文件最多只有一个）、INCR（增量AOF类型，它一般会在AOF重写的时候被执行，该文件可能存在多个，写操作命名就记在这个文件中）、HISTORY（历史AOF，由BASE和INCR类型的AOF文件演化而来，每次进行重写操作完成后，本次和之前对应的BASE和INCR类型的AOF文件都会变成HISTORY）。为了管理上述类型的文件，我们还有一个manifest（清单）文件来跟踪和管理这些AOF。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：HISTORY类型的AOF文件会被Redis自动删除，而且综上所述，Redis7中AOF文件的appendonly.aof仅仅是AOF文件名的前缀，可以在Redis配置文件中通过配置项appendfilename进行修改。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">8）AOF文件的恢复即将我们的AOF文件放在AOF文件产生的路径即可。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">9）若AOF文件出现了破损，我们可以使用&rdquo;redis-check-aof&nbsp; --fix &nbsp;INCR类型的aof文件路径&rdquo;。这个命令只修INCR类型的AOF，修其他的没必要（反正其他的AOF文件也不记录写命令）。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：redis-check-rdb和redis-check-aof都在路径/usr/local/bin中，且修复工作都要在对应的RDB或者AOF文件产生路径下工作。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">11）AOF文件通常比相同数据集的的等效RDB文件要大，并且恢复速度比RDB慢。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">12）AOF运行效率要比RDB慢，每秒同步回写策略效率比较好，不同步效率和RDB相同。</span></p>
<p align="left">&nbsp;</p>
<h3 align="left"><strong><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;">4.AOF进步实操（二）</span></strong></h3>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）由于AOF持久化是Redis不断将写命令记录到AOF文件中，随着Redis的不断进行，AOF文件会越来越大，为了解决这个问题，Redis新增了重写机制。重写机制即当AOF文件的大小超过所设定的峰值的时候，Redis会自动启动AOF文件的内容压缩，只保留可以恢复数据的最小指令集或者可以手动使用命令bgrewriteaof进行重写。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：禁用AOF的情况下我们仍然可以使用命令bgrewriteaof进行重写从而生成aof文件。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）Redis配置文件中配置重写机制的配置项如下</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">auto-aof-rewrite-percentage 100即根据上次重写后的aof文件大小，判断当前aof文件大小是否增长了1倍。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">auto-aof-rewrite-min-size 64mb即重写时满足的文件大小。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">以上两条配置项要同时满足才会触发重写机制。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）aof-use-rdb-preamble no即关闭AOF+RDB的混合使用。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">4）重写后的aof文件并不是在旧的文件上进行重写而是会生成一个新的aof文件。incr类型的aof文件中保存写指令，在进行一次重写后，incr类型的aof文件会重新生成一个并删除老的incr文件，此时incr类型的aof文件大小为0。重写压缩后的指令集由新产生的base类型的aof文件存储，同样的，老的base类型的aof文件会被删除。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">5）Redis配置文件关于AOF的优化项。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">no-appendfsync-on-rewrite即我们在aof重写的时候是否还允许对aof文件进行编写。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<h3 align="left"><span style="font-size: 14pt;"><strong><span style="font-family: 黑体, 'Heiti SC';">5.RDB和AOF的混合持久化（推荐使用）</span></strong></span></h3>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）RDB和AOF可以在同一个实例中一起使用，若开启了AOF，则Redis在启动的时候优先加载AOF，并且在Redis中AOF是主导者。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）若RDB和AOF在同一个实例中一起使用，重启Redis时只会加载aof文件不会加载rdb文件。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）RDB和AOF可以在同一个实例中一起使用，AOF作为主要的备份手段，RDB作为万一的手段。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：RDB更适合用于备份数据库（AOF因为不断变化的原因不太好备份），RDB做全量持久化，AOF做增量持久化。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">4）开启RDB和AOF的混合持久化是在Redis配置文件中将配置项aof-use-rdb-preamble配置为yes。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<h3 align="left"><strong><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;">6.纯缓存模式</span></strong></h3>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）纯缓存模式适用于高性能高并发的场景，即仅用Redis的缓存数据库功能，不关注它的备份（用其他方式实现备份），关闭所有的持久化功能（关闭RDB和AOF功能）。</span></p>
<p align="left"><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）关闭所有的持久化功能仅仅是关闭其自动触发的功能，但是手动触发持久化功能仍然是有效的。</span></p>
<p align="left">&nbsp;</p>