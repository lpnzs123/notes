<h1>一、双重for循环　</h1>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">　　Mysql中是没有for这个循环语句的，所以当我们想构建双重for循环的时候，就额外的麻烦，不仅要注意各种语法问题，还要对Mysql中的循环，存储过程了解深刻，下面是一个我构建出来的Mysql中的双重for循环，目的是同时显示出apply_against_audit表和approval_log中最大的id（主键）号</span></p>
<p><img src="https://img2023.cnblogs.com/blog/3171133/202312/3171133-20231225085312642-523358122.png" alt="" loading="lazy" /></p>
<p>&nbsp;　　<span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">在这段代码中，有以下几个需要注意的地方：</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">　　1.BEGIN..END中是可以嵌套BEGIN..END的，就像循环中可以嵌套循环。但是要注意，内层的BEGIN..END的END要加上顿号</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">　　2.HANDLER的定义，在一个BEGIN..END代码块中只能定义一个，并且HANDLER的定义要在打开游标代码之前，不然就会报错</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">　　3.当BEGIN..END代码块中嵌套了BEGIN..END代码块，那么内层的BEGIN..END代码块可以定义一个HANDLER，外层的BEGIN..</span><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">END</span><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">代码块也可以定义一个HANDLER，这跟注意事项2并不冲突</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">　　4.游标在一个BEGIN..END代码块中可定义的数量与HANDLER不同，前者想定义几个就定义几个，后者在一个BEGIN..END代码块</span><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">中仅仅可以定义一个</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">　　5.注意变量的申明在游标的申明之前，否则报错</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">&nbsp;</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">　　这里有一个问题，就是当我把内层BEGIN..END代码块中的变量申明语句提到外层的时候，输出的结果就不一样了</span></p>
<p><img src="https://img2023.cnblogs.com/blog/3171133/202312/3171133-20231225084959236-1461039741.png" alt="" loading="lazy" /></p>
<p>&nbsp;</p>
<p>&nbsp;　　<span style="font-family: 宋体, 'Songti SC';"><span style="font-size: 18px;">针对这个问题的分析也很简单，简要的给出下面的分析</span></span></p>
<p><img src="https://img2023.cnblogs.com/blog/3171133/202312/3171133-20231225085116169-499989215.png" alt="" loading="lazy" /></p>
<p>&nbsp;</p>
<p>&nbsp;　　<span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">在嵌套的BEGIN...END代码块中，hello:LOOP的FETCH放在了IF之前，这就会引发一个逻辑上的BUG。在每次外层LOOP（nihao）循环到内层LOOP（hello）的时候，都会重新的声明一次游标cursor02，这时候cursor02是从表最初的数据开始循环遍历的，而后进入到LOOP（hello）中。在LOOP（hello）中，我们将FETCH放在了IF之前，这就意味着，若是flag被设置为1了，重新被申明的游标cursor02在遍历到表第一条数据的时候就会退出，以后的每一次外层LOOP循环到内层LOOP都是这样。因为外层LOOP遍历一次，内层LOOP总是会完全遍历一次（内层LOOP的游标会把他遍历的表完全遍历），所以这个BUG，无论外层LOOP和内层LOOP对应的游标遍历的表记录数对比如何（哪个表的记录数大，哪个小），这个BUG都会发生</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">　　解决这个问题的方法也很简单，将flag的声明放在内嵌的BEGIN...END代码块（严格来说是放在外层LOOP当中，但是变量的声明必须在BEGIN...END代码块的最上方，不然就会报错，所以才说放在内嵌的BEGIN...END代码块中。在这里，内嵌的BEGIN...END代码块在外层的LOOP中，所以效果是一样的）中，或者调换LOOP（hello）中FETCH、IF的位置</span></p>
<p><span style="font-family: 宋体, 'Songti SC'; font-size: 18px;">　　附上源码：</span></p>
<pre class="language-armasm"><code>CREATE DEFINER=`root`@`localhost` PROCEDURE `findHighEmpInDept`(OUT aaid INT,OUT alid INT)
BEGIN
-- 	变量定义
		DECLARE flag INT DEFAULT 0;
		DECLARE done INT DEFAULT 0;
    DECLARE cursor01 cursor FOR SELECT aa_id FROM apply_against_audit;
		DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;
    OPEN cursor01;
		
		nihao: LOOP
			IF done = 1 THEN
				LEAVE nihao;
			END IF;
			FETCH cursor01 INTO aaid;
-- 	  新的代码块
			BEGIN
				DECLARE cursor02 cursor FOR SELECT al_id FROM approval_log;
				DECLARE CONTINUE HANDLER FOR NOT FOUND SET flag = 1;
				OPEN cursor02;
				
				hello: LOOP
					IF flag = 1 THEN 
						LEAVE hello;
					END IF;
					FETCH cursor02 INTO alid;
				END LOOP;
			END;
			
		END LOOP;
		CLOSE cursor01;
END</code></pre>
<p>&nbsp;</p>