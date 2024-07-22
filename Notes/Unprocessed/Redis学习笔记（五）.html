<h2><span style="font-family: 黑体, 'Heiti SC'; font-size: 18pt;"><strong>一、Redis事务</strong></span></h2>
<h3><strong><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;">1.基础概念</span></strong></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）Redis事务即可以一次性执行多个命令，本质是一组命令的集合。一个事务中的所有命令都会序列化，按顺序地串行化执行而不会被其他命令插入，不许加塞。换而言之，Redis事务即在一个队列中，一次性、顺序性、排他性的执行一系列命令。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）注意区分Redis事务和数据库事务</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第一、Redis事务中Redis命令执行是单线程架构，在执行完事务内所有的指令前是不可能再去同时执行其他客户端的请求的。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第二、Redis事务中无隔离级别的理论，不会有脏读、不可重复读等概念。因为Redis事务提交前任何指令都不会被实际执行。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第三、Redis事务不保证原子性，不保证所有事务中的指令同时成功或者同时失败，只有决定是否开始执行全部指令的能力，没有执行到一半进行回滚的能力。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">第四、Redis事务会保证一个事务内的命令依次执行，而不会被其他命令插入。</span></p>
<p>&nbsp;</p>
<h3><span style="font-size: 14pt;"><strong><span style="font-family: 黑体, 'Heiti SC';">2.与数据库事务类比</span></strong></span></h3>
<p><span style="font-size: 18px; font-family: 宋体, 'Songti SC';">Mysql事务是以begin开头，commit|rollback结尾，中间穿插sql语句形成事务。而Redis事务以multi开头，exec结尾，中间穿插Redis语句形成事务。</span></p>
<p>&nbsp;</p>
<h3><strong><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;">3.如何使用</span></strong></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）正常执行：以multi命令开始事务，然后开始书写事务命令（即一连串事务要执行的命令），最后以exec命令为止开始执行事务。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）放弃事务：以multi命令开始事务，然后开始书写事务命令，最后以discard命令为止放弃执行本次事务。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）整体放弃：若以multi命令开始事务后，加入到事务命令中的命令出错了，则整体事务被放弃执行（在使用exec命令后事务会整体执行失败）。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：在以multi命令开始事务后输入的每一个事务命令，都会被加入一个队列中等待执行。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">4）部分放弃：事务被执行后，事务命令中的命令有一些有问题，造成执行错误。此时并不会事务命令全体回滚，而是没有问题的事务命令执行成功，有问题的事务命令执行失败。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">5）watch监控：watch命令是一种乐观锁的实现。实现方式是用一个watch命令去监控我们的key（watch key1 key2....），然后去执行Redis事务。若这个key在我执行Redis命令期间没有被人动过，则皆大欢喜，事务成功。否则，本次事务执行后会返回null，告诉我这个事务被别人打断了，只好重新再来一次。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">6）unwatch命令：放弃watch监控。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1） 悲观锁即每次去拿数据的时候都会认为别人会在我修改数据时修改，所以在每次拿数据的时候都会上锁，这样别人想拿到这个数据的话就会被block，直到它拿到锁。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2） 乐观锁即每次去拿数据的时候都会认为别人不会在我修改数据时修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新我们拿到的数据。乐观锁的策略即提交的版本必须大于当前记录版本才能执行更新。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3） CAS类似于我们的watch监控，但又有不同。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">4） 若使用watch监控，则在每次Redis事务执行前都要使用watch命令开启监控。若在开启watch监控后发现被监控的key已经被修改了，可以使用unwatch命令解除对key的监控（直接在命令行输入unwatch即可）。这样，我们之后执行的事务变不会因为watch监控而失效，除非我们之后执行事务的方式是watch监控方式，则再分情况讨论。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">5） 一旦执行了事务（使用exec命令），则执行事务之前加的监控锁都会被取消掉。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">6） 当客户端连接丢失的时候（比如quit退出连接），所有的东西都会被取消监视。</span></p>
<p>&nbsp;</p>
<h2><strong><span style="font-family: 黑体, 'Heiti SC'; font-size: 18pt;">二、Redis管道</span></strong></h2>
<h3><span style="font-size: 14pt;"><strong><span style="font-family: 黑体, 'Heiti SC';">1.基本概念</span></strong></span></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）管道可以一次性的发送多条命令给服务端，服务端依次处理完毕后，通过一条响应一次性将结果返回，通过减少客户端与Redis的通信次数来实现降低数据包往返延时时间。其实现的原理是队列，保证了数据的顺序性。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）管道仅仅是将命令打包一次性发送，对整个Redis的执行不造成其他任何影响。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）管道即批处理命令变种优化措施，类似Redis的原生批命令（mget和mset）。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">注：RTT（Round Trip Time）即数据包往返于服务端客户端两端的时间。</span></p>
<h3><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;">2.管道实操</span></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）简单的批处理命令如mset|mget只能处理一种Redis数据类型的集合，但是管道可以处理多种数据类型命令组成的集合，我们需要将其写入一个文件中，如我们将命令集合写入一个cmd.txt文件中。然后利用Linux命令行窗口输入命令 &ldquo;cat cmd.txt | redis-cli -a xxxxxx(密码) --pipe&rdquo;即通过Linux管道将cmd.txt通过cat命令查询到的数据，在登录Redis后通过pipe管道传输到Redis服务端进行批处理。</span></p>
<h3><span style="font-family: 黑体, 'Heiti SC'; font-size: 14pt;">3.注意事项</span></h3>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">1）管道非原子性，原生的Redis批处理命令（如mset|mget）是原子性的。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">2）原生的批处理命令在Redis客户端中实现，但是管道需要Linux客户端和Redis客户端共同完成。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">3）管道是一次性将多条命令发送到服务器，而Redis事务是一条一条的发，事务只有在接收到exec命令后才会执行。执行事务的时候会阻塞其他命令的执行，而执行管道中的命令时并不会这样。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">4）管道中的命令依次执行，但是中途发生异常的命令执行并不会影响后面的命令正常执行。</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">5）不要一次性在管道中组装太多的命令，要适度，否则数据量过大客户端阻塞的时间可能过久，同时服务端此时也被迫回复一个队列答复, 以至于占用大量内存。</span></p>