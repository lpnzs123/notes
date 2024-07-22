<h2><strong><span style="font-family: 黑体, 'Heiti SC'; font-size: 18pt;">一、Redis集群的搭建</span></strong></h2>
<h3><strong><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;">1.Redis集群初始准备（三主三从）</span></strong></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">在三台虚拟机下的 /myredis目录下创建文件夹 cluster，可以使用命令&rdquo;mkdir -p /myredis/cluster&rdquo;，-p的含义即递归创建目录，若父目录不存在，则一并创建。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">保证三台虚拟机能够互相ping通。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">此Redis集群架构设计为A-B、C-D、E-F，其中A、C、E为master节点，B、D、F分别为其slave节点。实际集群搭建完毕后，master节点A的slave不一定是B，也可能是D或F，同理master节点C、E。</span></p>
<p>&nbsp;</p>
<h3><strong><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;">2.Redis集群的Redis配置文件的书写</span></strong></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">下面介绍其中一个Redis配置文件的书写，其余的配置文件仿照其书写即可</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）在/myredis/cluster建立对应的Redis配置文件(将Redis默认配置文件redis.conf复制到 /myredis/cluster目录下)，并修改相应的配置文件项</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2） Redis配置文件项daemonize改为yes（daemonize即是否开启后台运行Redis服务端）</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）Redis配置文件项 protected-mode改为no</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">4）Redis配置文件项 bind 0.0.0.0</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">5）Redis配置文件项port指定端口</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">6）Redis配置文件项dir指定当前工作目录（路径要用绝对路径）</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">7）Redis配置文件项pidfile指定pid文件的路径与名字</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">8）Redis配置文件项logfile指定log文件的路径与名字</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">9）Redis配置文件项 requirepass设置Redis的密码</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">10）Redis配置文件项dbfilename设置dump.rdb文件的名字</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">11）Redis配置文件项 appendonly yes开启AOF的功能</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">12）Redis配置文件项 appendfilename 设置AOF文件名字前缀</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">13）Redis配置文件项 materauth 设置访问master时的校验密码</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">14）Redis配置文件项 cluster-enabled yes开启集群</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">15） Redis配置文件项cluster-config-file设置集群节点相关信息的配置文件名，例&ldquo;cluster-config-file nodes-6381.conf&rdquo;。在集群建立联通成功后，此配置文件会自动创建</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">16）Redis配置文件项 cluster-node-timeout设置集群节点之间的超时时间，单位为毫秒</span></p>
<p>&nbsp;</p>
<h3><strong><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;">3.启动集群中所有的节点，并通过redis-cli命令为节点构建集群关系</span></strong></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">具体命令如下</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&ldquo;redis-cli -a&nbsp; Redis密码 &nbsp;--cluster create&nbsp; --cluster-replicas 1&nbsp; IP01:Port01&nbsp; IP02:Port02&nbsp;</span><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">IP03:Port03&nbsp; IP04:Port04&nbsp; IP05:Port05&nbsp; IP06:Port06&rdquo;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">此命令行表示这些IP与Port指定的Redis客户端，必须连成一片构成三主三从的集群。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">当看到如下所示的命令行提示，表示集群建立联通成功。</span></p>
<p><img src="https://img2023.cnblogs.com/blog/3171133/202309/3171133-20230903174148701-618861039.png" alt="" /></p>
<p>&nbsp;</p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）&rdquo;redis-cli -a&nbsp; Redis密码&rdquo;即以密码的形式登录Redis客户端。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）&rdquo;--cluster create&rdquo;即以集群的形式创建。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）&rdquo; --cluster-replicas 1&rdquo;即表示为每一个master创建一个slave节点。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">4）集群主从关系由Redis自动分配。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">5）若是Redis集群无法与一些节点取得联系，则会一直在Linux命令窗中提示&rdquo; Waiting for the cluster to join&rdquo;。若是防火墙均处于关闭状态，且虚拟机能够相互ping通，则可以排除通信被防火墙阻断和ip地址不能访问的问题。这时候可以考虑ip地址是否无法被虚拟机或Redis成功识别，即虚拟机或Redis相互无法识别对方的正确IP以至于相互无法沟通。这时候我们可以选择在Redis配置文件中声明节点在集群中的IP地址，对应的Redis配置文件项为&rdquo;cluster-announce-ip&rdquo;。</span></p>
<p>&nbsp;</p>
<h3><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;"><strong>4.随机登录一个Redis客户端查询集群状态</strong></span></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">使用cluster nodes的Redis客户端命令查询集群节点之间的关系，Redis客户端命令cluster info在某一个集群节点的Redis客户端上查询集群整体的大体信息。到此，Redis集群搭建成功。</span></p>
<p>&nbsp;</p>
<h2><span style="font-size: 18pt;"><strong><span style="font-family: 黑体, 'Heiti SC';">二、Redis集群的相关操作</span></strong></span></h2>
<h3><strong><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;">1.设置键值对与手动故障转移</span></strong></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）设置与取得键值对</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">由于Redis集群中槽位的限制，我们需要一种名为路由到位的机制帮助我们存储数据。这种机制若失效则会导致，我们在集群的某一个master节点上存储数据时，会发生存储失败的情况。这是因为，当Redis使用哈希槽分区算法时，并不一定会将键值对分配到我们连接到的master节点的槽位中，而是分配到其他master节点的槽位中。这就使得，我们键值对无法存储。当路由到位机制生效后，即使经过哈希槽分区算法后，我们所存储的键值对的槽位并不是我们目前登录的Redis客户端所管辖的，Redis也会自动的将我们的Redis客户端进行切换并对此键值对进行存储，这也是Redis客户端的自动重定向。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">为了防止路由到位机制的失效，我们在登录某一个节点的时候，需要加上 -c 参数，即&rdquo;redis-cli -a&nbsp; Redis密码 &nbsp;-p&nbsp; Redis服务端端口 &nbsp;-c&rdquo;，参数 -c 即代表集群中的节点经路由连接。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">综上所述，在登录到集群中节点的Redis客户端时，需要加上 -c 参数，才可以正常的访问集群。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：&ldquo;cluster keyslot k1&rdquo;即查询键为k1的键值对的所属槽位的槽位号。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）我们建立的是三主三从的Redis集群架构，在这六个节点中，我们有三对master-slave（一个master对应一个slave）。当其中一对master-slave的master宕机时，它的slave会自动上位成为master。宕机的旧master回归后，它会自动挂在新master底下当slave。以上的过程便是容错切换迁移，这个过程是自动的。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">我们也可以手动的故障转移（即进行节点从属调整），在slave的Redis客户端（即旧master）上输入&rdquo;cluster failover&rdquo;，旧master就会重新上位成为master，相应的，自动上位的新master会成为旧master的slave。</span></p>
<p>&nbsp;</p>
<h3><span style="font-size: 14pt;"><strong><span style="font-family: 黑体, 'Heiti SC';">2.Redis集群扩容步骤（三主三从扩容为四主四从）</span></strong></span></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）新建一主一从的两个Redis配置文件，配置好相关配置。再启动新建的一主一从两个实例，这两个实例分别为H（master）与I（slave）。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）选定一个集群中的节点作为介绍节点（介绍新的节点进入集群），然后利用以命令行将节点H加入集群，命令行为&ldquo;redis-cli &nbsp;-a&nbsp; Redis密码 &nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">--cluster add-node&nbsp; 节点H的IP:节点H的Redis服务端端口 &nbsp;介绍节点的IP:介绍节点的Redis服务端端口&rdquo;。若Linux命令窗口出现如下提示代表新节点加入集群成功。</span></p>
<p><img src="https://img2023.cnblogs.com/blog/3171133/202309/3171133-20230903174148733-237188777.png" alt="" /></p>
<p>&nbsp;</p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）使用Redis客户端命令&rdquo;cluster nodes&rdquo;或者Linux命令窗口命令&rdquo;redis-cli&nbsp; -a&nbsp; Redis密码 &nbsp;-cluster check&nbsp; 集群某一节点IP:集群某一节点服务端端口&rdquo;检查集群情况（后者可以显示出集群中16384个槽位在各个master节点上的分配情况，而前者无法显示，因此使用后者查询集群信息），可以发现集群中新加入的节点H未被分配槽位。那么，接下来的事就是重新分派槽位了。可以使用Linux命令&rdquo;redis-cli&nbsp; -a&nbsp; Redis密码 &nbsp;--cluster reshard&nbsp; 集群某一节点IP:集群某一节点服务端端口&rdquo;，键入命令行后会问你三个主要的问题。第一个问题，想要从16384个哈希槽中给多少槽位给新节点H。第二个问题，新节点H的节点ID。第三个问题，在Linux命令窗口中输入all或者done。all是所有集群节点都作为分配槽给新节点的源节点，done是自己选择从哪些节点上取槽，即输入了取槽的节点ID后，再输入done就会开始重新分派槽位。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：在槽位的重新分配中，第三个选择若是all，则会发现节点H从集群获取的分派的槽位并不连续，而是以前的集群master节点一人分一部分连续槽位给节点H，故节点H获得的槽位不是连续而是分段的，这样做的原因时彻底重新分配的成本太高了（彻底重新分配那么原集群拥有的key也要重新分配），所以一人匀一点是最好的（匀自己没有被key占了的槽位）。还有，既然节点H被分配了槽位，那么他就是master节点，接下来的步骤就是在master节点H下挂上slave节点I。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">4）在master节点H下挂上slave节点I，所使用的Linux命令窗口命令行为&rdquo;redis-cli&nbsp; -a&nbsp; Redis密码 &nbsp;--cluster add-node &nbsp;I节点IP:I节点端口 &nbsp;H节点IP:H节点端口 &nbsp;--cluster-slave&nbsp; --cluster-master-id&nbsp; H节点的节点ID 。&rdquo;</span></p>
<p>&nbsp;</p>
<h3><span style="font-size: 14pt;"><strong><span style="font-family: 黑体, 'Heiti SC';">3.Redis集群的缩容（四主四从缩容为三主三从）</span></strong></span></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">以下是将四主四从中的master节点H与slave节点I删除的步骤</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）先删slave节点I，查询slave节点I的节点ID，使用Linux命令窗口命令行&rdquo;redis-cli&nbsp; -a&nbsp; Redis密码 &nbsp;--cluster del-node&nbsp; 节点I的IP:节点I的Redis服务端端口 &nbsp;节点I的节点ID&rdquo;。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）将master节点H的槽位清空，并将其清出来的槽位还给指定的集群中的某个master节点，使用Linux命令窗口命令行&rdquo;redis-cli&nbsp; -a&nbsp; Redis密码 &nbsp;--cluster reshard&nbsp; 集群中的某一master节点的IP: 某一master节点的端口&rdquo;。键入命令行后会问你三个主要的问题。第一个问题，想要从16384个哈希槽中给多少槽位给指定的集群中的某个master节点（槽位数指定为master节点H拥有的槽位数即可）。第二个问题，指定的集群中的某个master节点的节点ID。第三个问题，在Linux命令窗口中输入all或者done。这里对于第三个问题，我们将节点H的节点ID输入到源节点（Source node），然后输入done。上述三个问题总结来说便是，我们将指定数量的槽位从源节点H中剥离，然后将剥离的槽位分配给指定的集群中的某个master节点。经过上述操作后，master节点H会成为我们指定的集群中的某一master节点的slave。当master节点H的槽位数被彻底清空（即剥离）后，使用命令行&rdquo;redis-cli&nbsp; -a&nbsp; Redis密码 &nbsp;-cluster check&nbsp; 集群某一节点IP:集群某一节点服务端端口&rdquo;检查集群情况，并不会显示master节点H的槽位数，但是master节点H仍然存在于集群中，因此我们需要删除它。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）删除节点H，查询slave节点H的节点ID，使用Linux命令窗口命令行&rdquo;redis-cli &nbsp;-a&nbsp; Redis密码 &nbsp;--cluster del-node&nbsp; 节点H的IP:节点H的Redis服务端端口 &nbsp;节点H的节点ID&rdquo;。</span></p>
<p>&nbsp;</p>
<h3><strong><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;">4.集群常用操作命令和CRC16算法分析</span></strong></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）不在同一个槽位的下的键值对无法使用批操作命令（mget，mset等）进行对多个键值对的操作。可以使用{ }来对键进行映射，映射方式为 key{映射名} 。这样使得映射名相同的键值对的值会放在同一个槽位中。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">例如：&rdquo;mset&nbsp; k1{x} v1 &nbsp;k2{x} v2 &nbsp;k3{x} v3&rdquo;即将k1、k2、k3的值都存放在映射名为x对应的槽位中。对应的，取值命令行为&rdquo; mget k1{x} k2{x} k3{x}&rdquo;。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）CRC16校验是调用c语言编写的方法，该方法的位置为cluster.c类下的keyHashSlot方法。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）Redis配置文件中有配置项&rdquo;cluster-require-full-coverage&rdquo;，该配置项的默认值为yes，意思为当集群（以三主三从为例）中，其中一对或多对（但不是所有）master和slave彻底宕机，集群不会对外提供服务，即配置项&rdquo;cluster-require-full-coverage&rdquo;的意思为集群是否完整才对外提供服务。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">4）&rdquo;cluster countkeysinslot 槽位编号&rdquo;即检查指定槽位编号对应的槽位是否被占用，返回值为1表示被占用，0表示未被占用。</span></p>