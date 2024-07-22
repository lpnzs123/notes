<h1>一、关于ES6箭头函数的理解</h1>
<p><img src="https://img2023.cnblogs.com/blog/3171133/202312/3171133-20231220161831060-1208898846.png" alt="" loading="lazy" /></p>
<span style="font-family: 宋体, 'Songti SC'; font-size: 18px;"><span style="color: #ff0000;">这里我认为有问题，问题如图中标注，问题理由如下图</span></span>
<p><img src="https://img2023.cnblogs.com/blog/3171133/202312/3171133-20231220161939260-1944745134.png" alt="" loading="lazy" /></p>
<p><span style="font-size: 18px; font-family: 宋体, 'Songti SC';"><span style="color: #ff0000;">　　第一个图中说handler对象init方法中，匿名函数方法块的this总是指向handler对象，但是在图二中又说明对象不构成单独的作用域，有一点前后的冲突，我的理解如下：</span></span></p>
<p><span style="font-size: 18px; font-family: 宋体, 'Songti SC';"><span style="color: #ff0000;">　　对象不构成单独的作用域，图一中的匿名函数中的this指向其实是指向handler对象的init方法（函数有自己的作用域）。又因为init方法处于handler对象之中（init</span></span><span style="font-size: 18px; font-family: 宋体, 'Songti SC';"><span style="color: #ff0000;">方法中的this即代表handler对象），所以</span></span><span style="font-size: 18px; font-family: 宋体, 'Songti SC';"><span style="color: #ff0000;">使得匿名函数的this，看起来就像</span></span><span style="font-size: 18px; font-family: 宋体, 'Songti SC';"><span style="color: #ff0000;">指向handler对象一样。因此，也可以使用handler对象之中的属性如handler对象属性id等。</span></span></p>
<p><span style="font-size: 18px; font-family: 宋体, 'Songti SC';"><span style="color: #ff0000;">　　图二中，匿名函数直接被对象包裹，这时候匿名函数的this指向的就是cat对象之外的对象，即全局对象，因为对象不构成单独的作用域。这个时候，无论如何调用cat</span></span><span style="font-size: 18px; font-family: 宋体, 'Songti SC';"><span style="color: #ff0000;">对象</span></span><span style="color: #ff0000; font-family: 宋体, 'Songti SC'; font-size: 18px;">cat.jumps()方法，lives的数字都不会减少。那如果我把cat对象写成如下形式呢？</span></p>
<p><span style="color: #ff0000; font-family: 宋体, 'Songti SC'; font-size: 18px;">　　　　const cat = {</span></p>
<p><span style="color: #ff0000; font-family: 宋体, 'Songti SC'; font-size: 18px;">　　　　　　lives:9,</span></p>
<p><span style="color: #ff0000; font-family: 宋体, 'Songti SC'; font-size: 18px;">　　　　　　jumps: function(){<br /></span></p>
<p><span style="color: #ff0000; font-family: 宋体, 'Songti SC'; font-size: 18px;">　　　　　　　　()=&gt;console.log(--this.lives)</span></p>
<p><span style="color: #ff0000; font-family: 宋体, 'Songti SC'; font-size: 18px;">　　　　　　}</span></p>
<p><span style="color: #ff0000; font-family: 宋体, 'Songti SC'; font-size: 18px;">　　　　}</span></p>
<p><span style="color: #ff0000; font-family: 宋体, 'Songti SC'; font-size: 18px;">　　结果是调用cat.jumps()方法lives值依然不会减少，因为cat.jumps()方法调用并不会使得箭头函数"()=&gt;console.log(--this.lives)"运行</span></p>
<p><span style="color: #ff0000; font-family: 宋体, 'Songti SC'; font-size: 18px;">　　在箭头函数一节，前文中有一个示例如下</span></p>
<p>&nbsp;</p>
<p><img src="https://img2023.cnblogs.com/blog/3171133/202312/3171133-20231220164554673-222720651.png" alt="" loading="lazy" /></p>
<p>&nbsp;　　<span style="color: #ff0000; font-family: 宋体, 'Songti SC'; font-size: 18px;">在这里可能会产生混淆与误解，让我们对于上面的理解产生误差，这里我们贴出一段话</span></p>
<p><img src="https://img2023.cnblogs.com/blog/3171133/202312/3171133-20231220164638013-29690501.png" alt="" loading="lazy" /></p>
<p>&nbsp;</p>