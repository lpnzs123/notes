# JavaScript的apply、call、bind方法



## 概述

这三个方法存在一定的迷惑性，因为他们的功能太过于相似却又有所区别。为了区分他们与理解他们的意图，确实需要下一番功夫。
<br/>

## 详解

从前端必备网站 MDN 上我们可以得知以上三种方法的完全体。

<br/>

> Function.prototype.apply(thisArg , argsArray);                    //apply方法
>
> Function.prototype.call(thisArg , ...args);                               //call方法
>
> Function.prototype.bind(thisArg , ...args);			     //bind方法

<br/>

然后我们来分别看看这三个方法的例子，首先是call，我把它放在第一个讲，因为我觉得它很有代表性。

<br/>

### Call方法

```javascript
function Product(name, price) {
  this.name = name;
  this.price = price;
}

function Food(name, price) {
  //这里调用了call方法
  Product.call(this, name, price);
  this.category = 'food';
}

console.log(new Food('cheese', 5).name);
// Expected output: "cheese"

```

第一眼看上去，这个方法到底是啥？怎么这么难理解？它到底做了什么？

<br/>

我们不考虑MDN对这个方法参数的专业解释，来用我们自己的语言描述这个方法。

<br/>

下面给出我们在上面代码块中调用call方法的部分。

```javascript
Product.call(this, name, price);
```

这句代码到底做了什么事情，首先我们从它的this入手，有这样一句话

<br/>

> 构造函数中this指向对象实例

<br/>

所以this指向的，就是`function Food(...)`构造方法的实例，call方法的第一个参数是谁我们就找到了。

接下来是call方法的第二个参数，注意call方法的完全体的第二个参数名字为 “argsArray” ，这个名字的意思是什么？参数数组，对了，所以 “name” 和 “price” 应该有一个统一的名称，参数数组中的参数。

现在我们解释了代码块中调用call方法部分的两个参数，那么，接下来的问题是call方法利用这两个参数做了什么事情呢？

<br/>

我们再把call方法的完全体拿过来看一看。

<br/>

> Function.prototype.apply(thisArg , argsArray);

<br/>

Function，Function是谁？在代码块调用call方法部分中，Function是Product，call方法会把参数传给Product。

注意，call方法一旦调用就立即执行对应的Function。

<br/>

经过上面的讨论我们基本上分析完了，下面给出我们的通俗理解。

<br/>

“我们将Product构造方法，绑定给了Food构造方法，并将Product构造方法的this指向Food构造方法的实例。也就是说我们执行Food构造方法时调用Product构造方法，这个调用是**Product构造方法的内容**在Food构造方法的**内部调用**”

<br/>

那这样说我直接在Food构造方法中调用Product构造方法就好了，为什么要用到call方法？
我们把这句话翻译成代码。

```javascript
function Food(name, price) {
  Product(name, price);
  ...
}
```

<br/>

把我们的通俗理解翻译成代码如下

```javascript
function Food(name, price) {
  //Product.call(this, name, price);和下面代码等价
  this.name = name;
  this.price = price;
  ...
}
```

看出区别了吗？两个代码块调用this的指向是不同的，`Product(name, price);`方法的this指向Product构造方法的实例，而后者代码块的this指向Food构造方法的实例。这样就使得`console.log(new Food('cheese', 5).name);`运行时，呈现两种完全不同的结果。

<br/>

前者代码块的`new Food('cheese', 5)`运行时会将参数直接传给`Product(name, price);`然后呢？然后就没有了！`Product(name, price);`将参数传给了Product构造方法，而Product构造方法的this指向他自己的实例，赋值当然也会赋给它自己的实例。

<br/>

这里附上Product构造方法的代码。

```javascript
function Product(name, price) {
  this.name = name;//赋值
  this.price = price;//赋值
}
```

<br/>

所以当运行`new Food('cheese', 5).name`时，返回的值是undefined，因为`new Food('cheese', 5)`产生的实例中，根本不含有name和price属性！

<br/>

因此，想让name和price的值赋给Food构造方法的实例，必须改变Product构造方法中this的指向，让Product构造方法的this指向Food构造方法的实例才行。

<br/>

call方法就为我们做了这件事，这也是call方法第一个参数的意义所在。

<br/>

### bind方法

```javascript
const module = {
  x: 42,
  getX: function () {
    return this.x;
  },
};

const unboundGetX = module.getX;
console.log(unboundGetX()); // The function gets invoked at the global scope
// Expected output: undefined

const boundGetX = unboundGetX.bind(module);
console.log(boundGetX());
// Expected output: 42
```

经过call方法的磨练，想必理解上述代码十分简单，但是要注意一件事情，即bind方法调用后，它对应的Function不会立即执行，要等我们去调用才执行。

<br/>

### apply方法

```javascript
const numbers = [5, 6, 2, 3, 7];

const max = Math.max.apply(null, numbers);

console.log(max);
// Expected output: 7

const min = Math.min.apply(null, numbers);

console.log(min);
// Expected output: 2

```

这里也不给出具体的解析了，同理call方法即可。注意，apply方法调用后，它对应的Function会立即执行。
