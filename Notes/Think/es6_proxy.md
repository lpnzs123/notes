# Es6_proxy代码案例讲解

## 概述

这是对于es6中proxy章节部分的几个例子做讲解，因为其晦涩难懂，所以单独抽出来做解释。

<br/>

## 例一

```javascript
var pipe = function (value) {
  var funcStack = [];
  var oproxy = new Proxy({} , {
    get : function (pipeObject, fnName) {
      if (fnName === 'get') {
        return funcStack.reduce(function (val, fn) {
          return fn(val);
        },value);
      }
      funcStack.push(window[fnName]);
      return oproxy;
    }
  });
  return oproxy;
}
var double = n => n * 2;
var pow    = n => n * n;
var reverseInt = n => n.toString().split("").reverse().join("") | 0;
pipe(3).double.pow.reverseInt.get; // 63
```

上述代码可以分成三部分，分别是代理部分，函数定义部分，和调用部分，我将针对这三部分按照最能让人理解的顺序进行讲解。

<br/>

### 函数定义部分

```javascript
var double = n => n * 2;
var pow    = n => n * n;
var reverseInt = n => n.toString().split("").reverse().join("") | 0;
```

double和pow函数意思很明确，不需要过多的解释，但是reverseInt函数可以稍微解释一下。

reverseInt函数做了两件事情。第一，将字符串反转，第二，将字符串类型变成数字类型。

其中`|0` 起的作用就是将字符串类型转成数字类型。

<br/>

### 代理部分

```javascript
var pipe = function (value) {
  var funcStack = [];
  var oproxy = new Proxy({} , {
    get : function (pipeObject, fnName) {
      if (fnName === 'get') {
        return funcStack.reduce(function (val, fn) {
          return fn(val);
        },value);
      }
      funcStack.push(window[fnName]);
      return oproxy;
    }
  });
  return oproxy;
}
```

首先，pipe是一个函数，我们调用pipe函数的时候会传入一个值value，最终函数调用的返回值是一个代理对象oproxy。

在pipe函数中我们定义了一个数组funcStack，用于存放我们函数定义部分定义的函数，这里需要注意一个问题，数组funcStack不是栈而是数组，例如`funcStack.push(1,2,3)`，我们输出的funcStack数组值为`[1,2,3]`，并不是`[3,2,1]`。这里做出说明是因为涉及到后面的函数调用顺序的问题，因此特意说明一下。

<br/>

代理对象oproxy，它给一个空对象 {} 设置了代理，其代理逻辑为，若是调用代理对象的属性名不为  "get" ，则执行下面的代码。

```javascript
funcStack.push(window[fnName]);
return oproxy;
```

`window[fnName]`根据字符串 fnName 来获取全局对象中具有相应名称的函数，并将其push到数组funcStack中，然后返回代理对象oproxy。

<br/>

若是调用代理对象的属性名为  "get" ，则执行下面的代码。

```javascript
return funcStack.reduce(function (val, fn) {
	return fn(val);
},value);
```

`funcStack.reduce`方法中回调函数`function (val, fn) {return fn(val);}`以value作为初始值赋值给val，fn为当前数组funcStack中正在调用的值（因为数组funcStack中全为函数，所以相当于调用函数），然后执行`fn(val)`所得的值赋值给累加器val用于下一轮的调用。

<br/>

### 调用部分

```javascript
pipe(3).double.pow.reverseInt.get;//63
```

综上所述，调用部分做的事情就是，以3作为初始值，分别调用double函数、pow函数和reverseInt函数，前一轮函数计算的值用在后一轮函数的计算之上，最终得出结果。

计算过程如下：

1. double(3) = 6
2. pow(6) = 36
3. reverseInt(36) = 63

<br/>

## 例二

```javascript
const dom = new Proxy({}, {
  get(target, property) {
    return function(attrs = {}, ...children) {
      const el = document.createElement(property);
      for (let prop of Object.keys(attrs)) {
        el.setAttribute(prop, attrs[prop]);
      }
      for (let child of children) {
        if (typeof child === 'string') {
          child = document.createTextNode(child);
        }
        el.appendChild(child);
      }
      return el;
    }
  }
});
const el = dom.div({},'Hello, my name is ',
                   dom.a({href: '//example.com'}, 'Mark'),
  				  '. I like:',
                   dom.ul(
    				{},dom.li({}, 'The web'),
                      dom.li({}, 'Food'),
                      dom.li({}, '…actually that\'s it')
				  )
);
document.body.appendChild(el);
```

<br/>

这个例子的解构就十分的痛苦了，这段代码我分为两部分，代理部分和调用部分。首先，我们解构代理部分。

<br/>

### 代理部分

```javascript
const dom = new Proxy({}, {
  get(target, property) {
    return function(attrs = {}, ...children) {
      const el = document.createElement(property);
      for (let prop of Object.keys(attrs)) {
        el.setAttribute(prop, attrs[prop]);
      }
      for (let child of children) {
        if (typeof child === 'string') {
          child = document.createTextNode(child);
        }
        el.appendChild(child);
      }
      return el;
    }
  }
});
```

<br/>

代理对象dom为一个空对象设置了代理，其代理的逻辑解构如下。

```javascript
get(target, property) {
    return function(attrs = {}, ...children) {...}
}
```

target为目标对象（ {} ），property为代理对象调用的属性名。当代理对象调用其对应的属性名时，需要传入两个参数，分别是attrs和children，参数attrs拥有默认值为 {} 。

<br/>

有点难以理解？没事，我们看看一下下面的代码。

```javascript
const dom = new Proxy({}, {
    get(target, property) {
    	return function(value) {
  			console.log(value);          
        }
	}
});

dom.nihao('haha');//haha
```

可以看到，`dom.nihao('haha');`输出的值为 'haha' ，这段代码究竟做了什么事情？

`dom.nihao`调用了dom代理对象的nihao属性，这时候get方法就会被调用，在`dom.nihao('haha')`中，我们给get方法里面return的方法传入了参数值 'haha'（注意，无论这个return的方法有没有函数名，参数值 'haha'都会被传入这个函数的形参中） 。这里要分清一件事，target参数在此时对应目标对象（ {} ），property参数在此时对应nihao属性，value参数在此时对应参数值 'haha' 。

最终，输出结果`dom.nihao('haha');//haha`

<br/>

相信经过这段代码的解释，上述内容也就不难理解了。

<br/>

接下来我们可以分析以下代码了。

```javascript
return function(attrs = {}, ...children) {
    //根据属性名创建el元素
	const el = document.createElement(property);
	for (let prop of Object.keys(attrs)) {
		el.setAttribute(prop, attrs[prop]);
	}
	for (let child of children) {
		if (typeof child === 'string') {
			child = document.createTextNode(child);
		}
		el.appendChild(child);
	}
	return el;
}
```

<br/>

我们用一个简单的例子去调用上述的方法

```javascript
dom.a({href: '//example.com'}, 'Mark');
```

此时的形参 attrs 值为`{href: '//example.com'}` ，形参children的值为 'Mark' 。我们先根据属性名创建el元素节点，然后对 attrs 进行第一个for循环的遍历，遍历出所有的键名，然后取出对应的键值，构成键值对作为el元素的属性名和属性值。

第二个for循环，我们开始遍历children。我们取出children（children为数组抑或是类似于数组的对象）中的每一项（在这里只有一项，就是 'Mark' ），然后对其进行类型检查，检查其是否为string类型，如果是，则创建一个文本节点child，之后添加到我们创建的元素节点el中。如果不是，则说明 child 为元素节点，直接添加到我们元素节点el中。

最后这个方法返回元素节点el。

<br/>

### 调用部分

```javascript
const el = dom.div({},'Hello, my name is ',
                   dom.a({href: '//example.com'}, 'Mark'),
  				  '. I like:',
                   dom.ul(
    				{},dom.li({}, 'The web'),
                      dom.li({}, 'Food'),
                      dom.li({}, '…actually that\'s it')
				  )
);
document.body.appendChild(el);
```

首先我们解释`dom.div(...)`这段代码的意思。

`dom.div(...)`这段代码调用了代理对象 dom 的 get 方法。因为这段代码想从 dom 这个代理对象身上取得 div 属性，那么必然就会触发对应的 get 方法。

<br/>

接下来是这一块看似复杂的代码块，我对他重新排一下版。

```javascript
dom.div(
    	//attrs
    	{},
        //children
        'Hello, my name is ',
    	dom.a({href: '//example.com'}, 'Mark'),
  		'. I like:',
        dom.ul(
             //attrs
    		{},
             //children
             dom.li({}, 'The web'),
             dom.li({}, 'Food'),
             dom.li({}, '…actually that\'s it')
         )
)
```

综上所述，这段代码创建了一个 div 标签（它没有属性），里面有 a 标签（有属性 href ，属性值为 '//example.com' ，有文本元素，值为  'Mark' ）、ul 标签和文本元素 '. I like:' 。

其中，ul 标签中又有三个 li 标签，分别有文本元素 'The web' 、 'Food' 和 '…actually that\'s it' ，最终效果如下。


![](https://img2024.cnblogs.com/blog/3171133/202401/3171133-20240111135049656-718530865.png)


最后的最后，`document.body.appendChild(el);`将我们创建的 el 元素加入到了页面的DOM文本中。
