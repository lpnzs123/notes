<h2><span style="font-size: 18pt; font-family: 黑体, 'Heiti SC';"><strong>一、Redis哨兵</strong></span></h2>
<h3><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;"><strong>1.概念</strong></span></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）Redis哨兵在没有使用Redis集群的时候能提供一种高可用的机制。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）Redis哨兵即设置一个吹哨人巡查监控后台master主机是否故障，若发生了故障则根据投票数自动的将某一个从库（slave）转换为新的主库（master），并继续对外服务。综上所述，Redis的哨兵作用有两个。第一，监控redis运行状态，包括master和slave。第二，当master宕机的时候，能够自动的将slave切换成新的master。简而言之，哨兵的作用就是一个无人值守的运维。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）Redis哨兵作用的进一步解释有以下四个</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第一、主从监控，即监控主从redis库运行是否正常。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第二、故障转移，即若master异常则会进行主从切换，让主从中的其中一个slave作为新的master。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第三、消息通知，即Redis哨兵可以将故障转移的结果发送给客户端。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第四、配置中心，即客户端通过连接哨兵来获得当前Redis服务的主节点地址。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">4）哨兵一般配成集群形式（一般配置三台哨兵），防止仅存在一台哨兵，当master和哨兵同时挂掉的时候，Redis哨兵机制的彻底失效。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：因为Redis哨兵是根据投票数自动的将某一个从库（slave）转换为新的主库（master），奇数的哨兵数有助于投票（投票行为在哨兵之间进行）。并且注意，哨兵作用仅仅涉及到自动监控和维护集群，不存放数据，只是吹哨人，所以哨兵的配置文件和Redis的配置文件，完全不一样。哨兵的默认配置文件的名字为sentinel.conf。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">5）哨兵是一种特殊的Redis进程，与Redis服务端平起平坐。</span></p>
<p>&nbsp;</p>
<h3><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;"><strong>2.哨兵配置文件sentinel.conf（这里我们在一台机器上配置三个哨兵，按理来说应该在三台机器上分别配置一个哨兵，在三台机器上分别配置一个哨兵的过程仿照下面流程并进行相应的改变即可）</strong></span></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）将哨兵默认配置文件sentinel.conf复制到与自定义Redis配置文件的同等级的目录下（/myredis）</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）哨兵配置文件项daemonize改为yes（daemonize即是否开启后台运行Redis服务端）</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）哨兵配置文件项 protected-mode改为no</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">4）哨兵配置文件项 bind 0.0.0.0</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">5）哨兵配置文件项port指定端口</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">6）哨兵配置文件项dir指定当前工作目录（路径要用绝对路径）</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">7）哨兵配置文件项pidfile指定pid文件的路径与名字</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">8）哨兵配置文件项logfile指定log文件的路径与名字</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">9）哨兵配置文件项 &ldquo;sentinel monitor&nbsp; master主机名 &nbsp;master主机IP&nbsp; Redis主机的端口 法定投票数&rdquo;，该配置文件项即设置需要被哨兵监控的master服务器。法定投票数（quorum）表示最少有几个哨兵认可客观下线，同意故障迁移的法定票数，简而言之，就是确认客观下线的最少的哨兵数量。（主机名自定义）</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：在哨兵集群中，一般需要多个哨兵互相沟通来确定他们关注的master是否真正宕机了。法定投票数（quorum）就是客观下线的一个依据，即至少有法定投票数那么多的哨兵认为master已经宕机了，才会真正的对这个master进行下线以及故障转移。因为，有些时候一个哨兵可能会因为网络拥堵而误以为某个master已经宕机了（例如哨兵自身网络问题导致无法连接master），此时master就没有出现故障，法定投票数的出现就保证了公平性和高可用，这就是客观下线。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">10）哨兵配置文件项&rdquo;sentinel auth-pass&nbsp; master主机名 &nbsp;master主机Redis的密码 &rdquo;即配置哨兵访问master主机的密码（主机名自定义）</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：哨兵的出现意味着一个master不可能永远都是master。那么，当角色互换的时候，也就是master变成slave 的时候，选举产生的新master也具有访问密码（Redis访问密码）。由master变成slave 的机子需要成为选举产生的新master的slave，那么，它就需要设置Redis配置文件中的masterauth配置项，以便于它访问新master。</span></p>
<p>&nbsp;</p>
<h3><strong><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;">3.哨兵配置文件的其他配置项的解释</span></strong></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）&rdquo;sentinel down-after-milliseconds&nbsp; master主机名&nbsp; 毫秒数&rdquo;即指定多少毫秒后，主节点（master）没有应答哨兵，此时哨兵主观上认为master已下线。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）&rdquo;sentinel parallel-syncs&nbsp; master主机名 &nbsp;数量&rdquo;即表示允许并行同步的slave个数。当master下线，哨兵从slave中选取一个称为master，那么，其他的slave就需要和这个新的master进行数据同步。这个配置文件项就是指定在新master和他的slave之间进行数据同步的最大slave数，直到所有的slave与新master数据同步，则故障转移结束。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）&rdquo;sentinel failover-timeout&nbsp; master主机名 &nbsp;毫秒数&rdquo;即故障转移的超时时间，进行故障转移时，若其时间超过设置的毫秒数，表示故障转移失败。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">4）&rdquo;sentinel notification-script&nbsp; master主机名 &nbsp;脚本路径&rdquo;即配置当某一事件发生时所需要执行的脚本。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">5）&rdquo;sentinel client-reconfig-script&nbsp; master主机名 &nbsp;脚本路径&rdquo;即用于配置在故障转移期间执行的客户端重配置脚本。当主服务器进行故障转移的时候，新的主服务器可能位于不同的服务器上，其主机名和端口可能会发生变化。这时，连接到旧主服务器的客户端可能需要重新配置以连接到新的主服务器。</span></p>
<p>&nbsp;</p>
<h3><span style="font-size: 14pt;"><strong><span style="font-family: 黑体, 'Heiti SC';">4.实操</span></strong></span></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）启动哨兵 可以使用命令行&rdquo;redis-sentinel&nbsp; 哨兵配置文件路径 &nbsp;--sentinel&rdquo;。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：哨兵启动成功后，会在其对应的配置文件中书写哨兵的基本信息（写在配置文件的最后），这时候可以去查看相应的信息。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）当master宕机的时候，哨兵会在slave中进行投票选举，这需要花一点时间。也就是说在选举的过程中，slave中的数据可能无法读取，直到新的master被选举出来，才可以进行读取数据的操作（数据读取不出来是暂时的，等会再试就行了）。使用命令info replication查询新主机是否已经被选举出来。新master选举出来后，旧master就变成了新master的slave。</span></p>
<p>&nbsp;</p>
<h3><strong><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;">5.附属问题</span></strong></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）在旧master宕机，新master被选举的过程中，可能会遇到以下两个问题</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第一、Error: Server closed the connection即服务器断开连接，这是在进行故障迁移，还没有成功，等一会（即新master被选举成功）就可以了。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第二、Error: Broken pipe即对端的管道已经断开，往往是远端把这个读写管道已经关闭了，导致无法再对这个管道进行读写操作。这个问题出现的时候其实对于服务端来说没有多少影响，Broken pipe这个问题也只会出现在往对端已经关闭的管道里写数据的情况下，这时候正确的做法应该是本端也close这个管道。在进行故障转移的时候出现这个问题，等一会再对数据进行操作就好了。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）哨兵在故障转移的时候会对对应的主从Redis配置文件进行修改，如旧master的配置文件加上配置文件项relicaof。哨兵添加的新的配置文件项会在Redis配置文件的最后，同时，哨兵会删除旧的配置文件项，如删除新master的replicaof配置文件项。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：与此同时，sentinel.conf这个配置文件的中的内容也会被修改（即哨兵的监控目标会随之调换），以适应新的主从关系。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3)哨兵可以同时监控多个master，在哨兵的配置文件中一行配一个就行了。</span></p>
<p>&nbsp;</p>
<h3><span style="font-size: 14pt;"><strong><span style="font-family: 黑体, 'Heiti SC';">6.哨兵运行流程及选举原理（以下步骤按顺序执行，模拟master宕机时哨兵运行流程及阐释其原理，整个过程无需人工干预，哨兵自行完成）</span></strong></span></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第一、主观下线（sdown）即单个哨兵自己主观上检测到的关于master的状态，从这个单个的哨兵来看，他向master发送心跳包后，在一定时间内没有收到合法的回复，就达到了主观下线的条件。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：同理适用于哨兵将slave添加到自己的监控范围，即哨兵可以认为slave主观下线。上述中发送心跳包后的一定时间内，这个一定时间可以在哨兵的配置文件中设置，其配置项为down-after-milliseconds。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第二、客观下线（odown）即一定数量的哨兵达成一致意见才可以认为一个master在客观上已经宕机了。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第三、哨兵中选出一个leader（领导者）进行从slave中选取新的master的工作。以三个哨兵为例，当master宕机，哨兵在进行投票客观下线旧的master后，需要再在slave中选出新的master。但是，这个过程并不是三个哨兵同时进行的，而是哨兵们从他们之中选出一个leader，由这个leader来指定slave中谁是新的master。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：leader是通过Raft算法选举出来的，Raft算法讲究一个先到先得，并且符合一条规则，即哨兵A向哨兵B发送成为leader的申请，如果哨兵B没有同意过其他哨兵的申请，则会同意哨兵A的申请。假设存在三台哨兵A、B、C，哨兵A先向B和C发出成为leader的申请，因为哨兵B和C没有收到除他们自己以外其他哨兵的申请，所以哨兵A获得两票。然后哨兵B向哨兵A和C发送申请，哨兵C已经同意了哨兵A，所以哨兵C会拒绝哨兵B，但是因为哨兵A没有同意过任何哨兵，所以它会同意哨兵B，哨兵B一票。同理哨兵C零票。最终leader为哨兵A。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第四、选出的leader指定一个slave成为新的master。选定过程中的评判标准有三个，依次进行评判。若有一个slave在按顺序进行的其中一个评判标准中胜出则选定结束，新master为该slave，并且不再进行后续标准评判。若在同一个标准下都一样，则使用下一个评判标准进行选定。评判标准依次为优先级（高则胜出）、复制偏移量（大则胜出）、Run ID（每一个redis实例运行起来都会有一个，其ASCII码最小的为新master）。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：选出新master的规则是在剩余slave节点都健康的前提下选定的。在redis配置文件中，有配置项slave-priority或者replica-priority，其值（数字）越小，优先级越高。在redis复制中，由于网络波动可能导致master和salve之间的数据不一致。例如，master A手下有slave B 和slave C，因为网络波动导致slave B 和slave C可能数据不一致，即slave B比slave C抄的少或多。这时，抄的少就是复制偏移量小，反之，复制偏移量大。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第五、在leader选出slave后，leader会做两个步骤。对选出的slave执行slaveof no one操作，使其变成新的master，而后再对其他slave（此时旧master处于宕机状态）发送命令，让其他的slave成为新master的slave。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第六、最后，宕机的旧master上线后，leader会让旧master降级为slave，并将旧master设置为新master的slave。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：哨兵会对相应配置文件进行修改，让上述过程从暂时变成永久。</span></p>
<p>&nbsp;</p>
<h3><strong><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;">7.哨兵使用建议</span></strong></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）哨兵应设置多个，为集群，数量为奇数。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）各个哨兵节点所处的计算机硬件最好一致。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）若哨兵节点部署在Docker等容器中，注意端口的正确映射。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">4）哨兵集群 + 主从复制并不能保证数据零丢失。例如，一主二从，当master宕机，在哨兵选出新master之前无法对redis进行写操作。那么，这个时间段中就会丢失数据。</span></p>