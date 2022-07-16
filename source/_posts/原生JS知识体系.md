---
title: 原生js知识体系总结
date: {{ date }}
tags: JS
cover: https://ppoffice.github.io/hexo-theme-icarus/gallery/covers/vector_landscape_1.svg
excerpt: A complete sum for vanilla javascript 
toc: true
---
# 原生JS知识体系

## 数据类型-概念

### 1.JS原始数据类型有哪些？引用数据类型有哪些？

在 JS 中，存在着 7 种原始值，分别是：

- boolean
- null
- undefined
- number
- string
- symbol
- bigint

引用数据类型: 对象Object（包含普通对象-Object，数组对象-Array，正则对象-RegExp，日期对象-Date，数学函数-Math，函数对象-Function）

### 2.null是对象吗？为什么？

结论: null不是对象。

解释: 虽然 typeof null 会输出 object，但是这只是 JS 存在的一个悠久 Bug。在 JS 的最初版本中使用的是 32 位系统，为了性能考虑使用低位存储变量的类型信息，000 开头代表是对象然而 null 表示为全零，所以将它错误的判断为 object 。



### 3. 如何理解BigInt?

#### 什么是bigint

> BigInt是一种新的数据类型，用于当整数值大于Number数据类型支持的范围时。这种数据类型允许我们安全地对`大整数`执行算术操作，表示高分辨率的时间戳，使用大整数id，等等，而不需要使用库。

#### 为什么需要BigInt?

在JS中，所有的数字都以**双精度64位浮点**格式表示，那这会带来什么问题呢？

这导致JS中的Number无法精确表示非常大的整数，它会将非常大的整数四舍五入，确切地说，JS中的Number类型只能安全地表示-9007199254740991(-(2^53-1))和9007199254740991（(2^53-1)），任何超出此范围的整数值都可能失去精度。

#### 如何创建并使用BigInt？

要创建BigInt，只需要在数字末尾追加n即可。

```js
console.log( 9007199254740995n );    // → 9007199254740995n	
console.log( 9007199254740995 );     // → 9007199254740996
```

另一种创建BigInt的方法是用BigInt()构造函数、

```js
BigInt("9007199254740995");    // → 9007199254740995n
```

简单使用如下:

```js
10n + 20n;    // → 30n	
10n - 20n;    // → -10n	
+10n;         // → TypeError: Cannot convert a BigInt value to a number	
-10n;         // → -10n	
10n * 20n;    // → 200n	
20n / 10n;    // → 2n	
23n % 10n;    // → 3n	
10n ** 3n;    // → 1000n	

const x = 10n;	
++x;          // → 11n	
--x;          // → 9n
console.log(typeof x);   //"bigint"
```

#### 值得警惕的点

1. BigInt不支持一元加号运算符, 这可能是某些程序可能依赖于 + 始终生成 Number 的不变量，或者抛出异常。另外，更改 + 的行为也会破坏 asm.js代码。
2. 因为隐式类型转换可能丢失信息，所以不允许在bigint和 Number 之间进行混合操作。当混合使用大整数和浮点数时，结果值可能无法由BigInt或Number精确表示。

```js
10 + 10n;    // → TypeError
复制代码
```

1. 不能将BigInt传递给Web api和内置的 JS 函数，这些函数需要一个 Number 类型的数字。尝试这样做会报TypeError错误。

```js
Math.max(2n, 4n, 6n);    // → TypeError
复制代码
```

1. 当 Boolean 类型与 BigInt 类型相遇时，BigInt的处理方式与Number类似，换句话说，只要不是0n，BigInt就被视为truthy的值。

```js
if(0n){//条件判断为false

}
if(3n){//条件为true

}
复制代码
```

1. 元素都为BigInt的数组可以进行sort。
2. BigInt可以正常地进行位运算，如|、&、<<、>>和^

## 数据类型-检测

### 1. typeof 是否能正确判断类型？

对于**原始类型**来说，**除了 null** 都可以调用typeof显示正确的类型。

```js
typeof 1 // 'number'
typeof '1' // 'string'
typeof undefined // 'undefined'
typeof true // 'boolean'
typeof Symbol() // 'symbol'

typeof null // 'object'-> 错误！！
```

但对于引用数据类型，**除了函数**之外，都会显示"object"。

```js
typeof [] // 'object'
typeof {} // 'object'
typeof console.log // 'function'
```

因此采用typeof判断对象数据类型是不合适的，采用instanceof会更好，instanceof的原理是基于**原型链**的查询，只要处于原型链中，判断永远为true

```js
const Person = function() {}
const p1 = new Person()
p1 instanceof Person // true

var str1 = 'hello world'
str1 instanceof String // false

var str2 = new String('hello world')
str2 instanceof String // true
```

### 2. instanceof能否判断基本数据类型？

能。比如下面这种方式:

```js
class PrimitiveNumber {
  static [Symbol.hasInstance](x) {
    return typeof x === 'number'
  }
}
console.log(111 instanceof PrimitiveNumber) // true
复制代码
```

如果你不知道Symbol，可以看看[MDN上关于hasInstance的解释](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FWeb%2FJavaScript%2FReference%2FGlobal_Objects%2FSymbol%2FhasInstance)。

其实就是**自定义instanceof行为**的一种方式，这里将原有的instanceof方法重定义，换成了typeof，因此能够判断基本数据类型。

### 3. 能不能手动实现一下instanceof的功能？

核心: 原型链的向上查找。

```js
function myInstanceof(left, right) {
    //基本数据类型直接返回false
    if(typeof left !== 'object' || left === null) return false;
    //getProtypeOf是Object对象自带的一个方法，能够拿到参数的原型对象
    let proto = Object.getPrototypeOf(left);
    while(true) {
        //查找到尽头，还没找到
        if(proto == null) return false;
        //找到相同的原型对象
        if(proto == right.prototype) return true;
        proto = Object.getPrototypeOf(proto);
    }
}
```

### 4. Object.is和===的区别？

Object在严格等于的基础上修复了一些特殊情况下的失误，具体来说就是+0和-0，NaN和NaN。 源码如下：

```js

function is(x, y) {
  if (x === y) {
    //运行到1/x === 1/y的时候x和y都为0，但是1/+0 = +Infinity， 1/-0 = -Infinity, 是不一样的
    return x !== 0 || y !== 0 || 1 / x === 1 / y;
  } else {
    //NaN===NaN是false,这是不对的，我们在这里做一个拦截，x !== x，那么一定是 NaN, y 同理
    //两个都是NaN的时候返回true
    return x !== x && y !== y;
  }
}
```



## 数据类型-转换

### 1. [] == ![]结果是什么？为什么？

解析:

== 中，左右两边都需要转换为数字然后进行比较。

[]转换为数字为0。

![] 首先是转换为布尔值，由于[]作为一个引用类型转换为布尔值为true,

因此![]为false，进而在转换成数字，变为0。

0 == 0 ， 结果为true

### 2. JS中类型转换有哪几种？

JS中，类型转换只有三种：

- 转换成数字
- 转换成布尔值
- 转换成字符串

转换具体规则如下:

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/10/20/16de9512eaf1158a~tplv-t2oaga2asx-zoom-in-crop-mark:1304:0:0:0.awebp)

### 3. == 和 ===有什么区别？

> ===叫做**严格相等**，是指：左右两边不仅**值**要相等，**类型**也要相等，例如'1'===1的结果是false，因为一边是string，另一边是number。 

==不像===那样严格，对于一般情况，只要值相等，就返回true，但==还涉及一些类型转换，它的转换规则如下：

- 两边的类型是否相同，相同的话就比较值的大小，例如1==2，返回false
- 判断的是否是null和undefined，是的话就返回true
- 判断的类型是否是String和Number，是的话，把String类型转换成Number，再进行比较
- 判断其中一方是否是Boolean，是的话就把Boolean转换成Number，再进行比较
- 如果其中一方为Object，且另一方为String、Number或者Symbol，会将Object转换成字符串，再进行比较

```js
console.log({a: 1} == true);//false
console.log({a: 1} == "[object Object]");//true
```

### 4. 对象转原始类型是根据什么流程运行的？

对象转原始类型，会调用内置的[ToPrimitive]函数，对于该函数而言，其逻辑如下：

1. 如果有Symbol.toPrimitive()方法，优先调用再返回
2. 调用valueOf()，如果转换为原始类型，则返回
3. 调用toString()，如果转换为原始类型，则返回
4. 如果都没有返回原始类型，会报错

```js
var obj = {
  value: 3,
  valueOf() {
    return 4;
  },
  toString() {
    return '5'
  },
  [Symbol.toPrimitive]() {
    return 6
  }
}
console.log(obj + 1); // 输出7
```

### 5. 如何让if(a == 1 && a == 2)条件成立？

其实就是上一个问题的应用。

```js
var a = {
  value: 0,
  valueOf: function() {
    this.value++;
    return this.value;
  }
};
console.log(a == 1 && a == 2);//true
```

### 6 隐式类型转换

https://chinese.freecodecamp.org/news/javascript-implicit-type-conversion/

## 闭包

### 什么是闭包？

> MDN 对闭包的定义为：闭包是指那些能够访问自由变量的函数。
>
> （其中自由变量，指在函数中使用的，但既不是函数参数arguments也不是函数的局部变量的变量，其实就是另外一个函数作用域中的变量。）
>
> 红宝书(p178)上对于闭包的定义：闭包是指有权访问另外一个函数作用域中的变量的函数，

**闭包是一个特殊的对象** **它由两部分组成，执行上下文A以及在A中创建的函数B**。

**当B执行时，如果访问了A中的变量对象，那么闭包就会产生。**

**在大多数理解中，包括许多著名的书籍、文章里都以函数B的名字代指这里生成的闭包。而在chrome中，则以执行上下文A的函数名代指闭包。**

### 闭包产生的原因?

首先要明白作用域链的概念，其实很简单，在ES5中只存在两种作用域————全局作用域和函数作用域，`当访问一个变量时，解释器会首先在当前作用域查找标示符，如果没有找到，就去父作用域找，直到找到该变量的标示符或者不在父作用域中，这就是作用域链`，值得注意的是，每一个子函数都会拷贝上级的作用域，形成一个作用域的链条。 比如:

```js
var a = 1;
function f1() {
  var a = 2
  function f2() {
    var a = 3;
    console.log(a);//3
  }
}
```

在这段代码中，f1的作用域指向有全局作用域(window)和它本身，而f2的作用域指向全局作用域(window)、f1和它本身。而且作用域是从最底层向上找，直到找到全局作用域window为止，如果全局还没有的话就会报错。就这么简单一件事情！

闭包产生的本质就是，**当前环境中存在指向父级作用域的引用**。还是举上面的例子:

```js
function f1() {
  var a = 2
  function f2() {
    console.log(a);//2
  }
  return f2;
}
var x = f1();
x();
```

这里x会拿到父级作用域中的变量，输出2。因为在当前环境中，含有对f2的引用，f2恰恰引用了window、f1和f2的作用域。因此f2可以访问到f1的作用域的变量。

那是不是只有返回函数才算是产生了闭包呢？

回到闭包的本质，我们只需要让父级作用域的引用存在即可，因此我们还可以这么做：

```js
var f3;
function f1() {
  var a = 2
  f3 = function() {
    console.log(a);
  }
}
f1();
f3();
```

### 闭包有哪些表现形式?

在真实的场景中，究竟在哪些地方能体现闭包的存在？

1. 返回一个函数。刚刚已经举例。
2. 作为函数参数传递

```js
var a = 1;
function foo(){
  var a = 2;
  function baz(){
    console.log(a);
  }
  bar(baz);
}
function bar(fn){
  // 这就是闭包
  fn();
}
// 输出2，而不是1
foo();
```

3. 在定时器、事件监听、Ajax请求、跨窗口通信、Web Workers或者任何异步中，只要使用了**回调函数**，实际上就是在使用闭包。
4. IIFE(立即执行函数表达式)创建闭包, 保存了`全局作用域window`和`当前函数的作用域`，因此可以全局的变量。

### 如何解决下面的循环输出问题？

```js
for(var i = 1; i <= 5; i ++){
  setTimeout(function timer(){
    console.log(i)
  }, 0)
}
```

为什么会全部输出6？如何改进，让它输出1，2，3，4，5？(方法越多越好)

因为setTimeout为宏任务，由于JS中单线程eventLoop机制，在主线程同步任务执行完后才去执行宏任务，因此循环结束后setTimeout中的回调才依次执行，但输出i的时候**当前作用域没有**，**往上一级再找**，发现了i,此时循环已经结束，i变成了6。因此会全部输出6。

解决方法：

1、利用IIFE(立即执行函数表达式)当每次for循环时，把此时的i变量传递到定时器中

```js
for(var i = 1;i <= 5;i++){
  (function(j){ // 创建了新的函数作用域，当setTimeout执行timer时，向上寻找作用域链会找到这个变量
    setTimeout(function timer(){
      console.log(j)
    }, 0)
  })(i)
}
```

2、给定时器传入第三个参数, 作为timer函数的第一个函数参数

```js
for(var i=1;i<=5;i++){
  setTimeout(function timer(j){
    console.log(j)
  }, 0, i)
}
```

3、使用ES6中的let

```js
for(let i = 1; i <= 5; i++){ // 本质上是创建了块级作用域
  setTimeout(function timer(){
    console.log(i)
  },0)
}

```

**let**使JS发生革命性的变化，让JS由**函数作用域**变为了**块级作用域**，用let后**作用域链不复存在**。代码的**作用域以块级为单位**，以上面代码为例:

```js
// i = 1
{
  setTimeout(function timer(){
    console.log(1)
  },0)
}
// i = 2
{
  setTimeout(function timer(){
    console.log(2)
  },0)
}
// i = 3
...
```

因此能输出正确的结果。

### 闭包的使用场景

#### 1. 模仿块级作用域

```javascript
for(var i = 0; i < 5; i++) {
    (function(j){
        setTimeout(() => {
            console.log(j);
        }, j * 1000);
    })(i)
}
```

#### 2. 模块化（module pattern）

通过函数或者IIFE实现模块化，将成员变量（函数内部定义的变量、参数等）私有化，对外提供公共方法操控这些变量。

**每一个JS模块都可以认为是一个独立的作用域，当代码执行时，该词法作用域创建执行上下文，如果在模块内部，创建了可供外部引用访问的函数时，就为闭包的产生提供了条件，只要该函数在外部执行访问了模块内部的其他变量，闭包就会产生。**



**闭包模块的第一种写法：**

```js
// HH: 闭包类的第一种写法
var PeopleClass = function () {
    var age = 18
    var name = 'HAVENT'

    // 闭包返回公开对象
    return {
        getAge: function () {
            return age
        },
        getName: function () {
            return name
        }

    }
}

// HH: 闭包类的第一种写法的调用
var people = new PeopleClass()
console.log(people.getAge())
console.log(people.getName())
```

**闭包模式的第二种写法**

```js
// HH: 闭包类的第二种写法
var PeopleClass = function () {
    var main = {}
    var age = 18
    var name = 'HAVENT'
    
    main.getAge = function () {
        return age
    }
    
    main.getName = function () {
        return name
    }

    // 闭包返回公开对象
    return main
}

// HH: 闭包类的第二种写法的调用
var people = new PeopleClass()
console.log(people.getAge())
console.log(people.getName())
```

**闭包模式的自动实例化对象的写法(IIFE、单例)**

```js
// HH: 闭包类的自动实例化对象的写法
var we = we || {}
we.people = (function () {
    var age = 18
    var name = 'HAVENT'

    function getPeopleAge () {
        return age
    }

    function getPeopleName() {
        return name
    }

    // 闭包返回公开对象
    return {
        getAge: function () {
            return getPeopleAge()
        },
        getName: function () {
            return getPeopleName()
        }
    }
})()

// HH: 闭包类的自动实例化对象的写法的调用
console.log(we.people.getAge())
console.log(we.people.getName())
```

#### 3. React hooks中的闭包

https://juejin.cn/post/6844904006079217672

```js
// state.js
let state = null;

export const useState = (value: number) => {
  // 第一次调用时没有初始值，因此使用传入的初始值赋值
  state = state || value;

  function dispatch(newValue) {
    state = newValue;
    // 假设此方法能触发页面渲染
    render();
  }

  return [state, dispatch];
}
```

在其他模块中引入并使用。

```jsx
import React from 'react';
import {useState} from './state';

function Demo() {
  // 使用数组解构的方式，定义变量
  const [counter, setCounter] = useState(0);

  return (
    <div onClick={() => setCounter(counter + 1)}>hello world, {counter}</div>
  )
}

export default Demo();
```

根据闭包的特性，state模块中的state变量，会持久存在。因此当Demo函数再次执行时，我们也能获取到上一次Demo函数执行结束时state的值。

**这就是React Hooks能够让函数组件拥有内部状态的基本原理。**

#### 4. webpack中的模块化

#### 5. node.js的模块化



### 闭包的优缺点

**优点：**

> 变量长期驻扎在内存中
> 避免全局变量的污染
> 可以定义私有属性和私有方法

**缺点：**

> 常驻内存 会增大内存的使用量 使用不当会造成内存泄露
> 可以改变父函数内部变量的值

闭包有一个非常严重的问题，那就是内存浪费问题，这个内存浪费不仅仅因为它常驻内存，更重要的是，对闭包的使用不当会造成无效内存的产生

#### 使用闭包造成内存泄漏的场景

https://segmentfault.com/a/1190000039132414

## V8中的闭包及内存泄漏

链接：https://juejin.cn/post/7079995358624874509

网络上对闭包的解释基本上都和 MDN 大同小异，“闭包就是访问了自由变量的函数”，其实这是为了大众方便理解而给出的错误结论（即使是这样似乎也有许多人无法理解闭包）

对于闭包产生的内存泄漏，网络中流传的大多数说法都是：“因为子函数执行时父函数的执行上下文已经退出执行上下文栈，但是由于子函数作用域链的引用导致父函数的 `活动对象AO` 无法被销毁”导致的。

其实上面的这两个广为流传的方法都是错误的，下面我将为你介绍真正的闭包和其内存泄漏的产生原理。

### 作用域链 `[[Scopes]]`

全局代码存储其变量的地方叫做变量对象（VO），函数存储其变量的叫活动对象（AO），VO 和 AO 都是在预编译时确定其内容，然后在代码运行时被修改值。

> ⚠注意：VO和AO都是在es1、3中才存在的概念，在现在的 es5+ 中已经不存在VO和AO的概念，取而代之的是一个叫做 词法环境（Lexical environment） 的东西，这里搬出来单纯是为了方便大家理解，后面我也将用 词法环境 代替 AO 和 VO 等概念。

关于词法环境可以看看这篇文章 [浅析JavaScript词法环境](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F159369973)
 对于 `作用域链`、`AO`、`VO` 如果想详细了解可以 [看这里](https://juejin.cn/post/6844903473528602637)

每一个函数都有一个 `[[Scopes]]` 属性，其存储的是这个函数运行时的作用域链，除了当前函数的 词法环境LE，作用域链的其他部分都会在其父函数预编译时添加到函数的 `[[Scopes]]` 属性上（因为父函数也需要预编译后才能确定自己的 `函数词法环境(function environment)`），所以 js 的作用域是词法作用域。

```js
// 1: 全局词法环境global.LE = {t,fun}
let t = 111
function fun(){
    // 3: fun.LE = {a,b,fun1}
    let a = 1
    let b = 2
    function fun1() {
        // 5: fun1.LE = {c}
        let c = 3
    }
    // 4: fun1.[[Scopes]] = [global.LE, fun.LE]
}
// 2: fun.[[Scopes]] = [global.LE]
fun()
```

上面代码在 `fun()` 被调用前，会立即预编译 `fun` 函数，这一步会得到 `fun` 的词法环境（LE），然后运行 fun 函数，在执行到 `let a = 1` 的时候，会将变量对象到 a 属性改成 1。后面也是一样

`[[Scopes]]` 就像一个数组一样，<u>每一个函数的 `[[Scopes]]` 中都存在当前函数的 LE 和上级函数的 `[[Scopes]]`</u>。在函数运行时会优先取距离当前函数 LE 近的变量值，这就是作用域的就近原则。

#### V8中闭包的实现机制

上面介绍的 `[[Scopes]]` 可能就是大家熟知的，这在以前是对的。

其实每一个 `词法作用域` 都会有一个 `outer` 属性指向其上级 `词法作用域`，根据这个 `outer` 链路完全可以构成作用域链，为什么要多此一举弄一个 `Closure` 出来呢？

这就是涉及到闭包和内存泄漏问题，如果单纯的通过 outer 链路来实现作用域链，那么存在一个闭包时，就会导致整个作用域链中的**所有** 词法环境 都无法回收，但是此时如果我们只使用了父级词法环境中的一个变量，而 V8 为了让我们能使用这个一个变量付出如此大的内存代价，很显然是不值得的。而 `[[Scopes]]` + `Closure` 就是他们的解决方案。

所以现在的 V8 中已经发生了改变（Chrome 中已经可以看到这些变化），在为一个函数绑定词法作用域时，<u>并不会粗暴的直接把父函数的 LE 放入其 `[[Scopes]]` 中</u>，而是会<u>分析这个函数中会使用父函数的 LE 中的哪些变量</u>，而这些可能<u>会被使用到的变量会被存储在一个叫做 `Closure` 的对象中</u>，每一个函数都有且只有一个 `Closure` 对象，最终这个 `Closure `将会代替父函数的 LE 出现在子函数的 `[[Scopes]]` 中

> 网络上的说法是：父函数的 AO 直接会被放入子函数的 `[[Scopes]]` 中，也没有提到 LE 和 `Closure` 对象，很明显这放在现在来看是不对的，当前后面我会给出例子证明。

### 闭包对象 `Closure`

在V8中每一个函数执行前都会进行预编译，预编译阶段都会执行3个重要的字节码

1. CreateFunctionContext 创建函数执行上下文
2. PushContext 上下文入栈
3. CreateClosure 创建函数的闭包对象

也就是说，每一个函数执行前都会创建一个闭包，无论这个闭包是否被使用，那么闭包中的内容是什么？如何确定其内容？

`Closure` 跟 `[[Scopes]]` 一样会在函数**预编译**时被确定，**区别**是当前函数的 `[[Scopes]]` 是在其**父函数**预编译时确定， 而 `Closure` 是在**当前函数**预编译时确定（在当前函数执行上下文创建完成入栈后就开始创建闭包对象了）。

当 V8 预编译一个函数时，如果遇到内部函数的定义不会选择跳过，而是会快速的扫描这个内部函数中使用到的本函数 LE 中的变量，然后将这些变量的引用加入 `Closure` 对象。再来为这个内部函数函数绑定 `[[Scopes]]` ，并且使用当前函数的 `Closure` 作为内部函数 `[[Scopes]]` 的一部分。

> 注意：每一次遇到内部声明的函数/方法时都会这么做，无论其内部函数/方法的声明嵌套有多深，并且他们使用的都是同一个 `Closure` 对象。并且这个过程 **是在预编译时进行的而不是在函数运行时**。

```js
// 1: global.LE = {t,fun}
var t = 111
// 2: fun.[[Scopes]] = [global.LE]
function fun(){
    // 3: fun.LE = {a,b,c,fun1,obj}，并创建一个空的闭包对象fun.Closure = {}
    let a = 1,b = 2,c = 3
    // 4: 遇到函数，解析到函数会使用a，所以 fun.Closure={a:1} (实际没这么简单)
    // 5: fun1.[[Scopes]] = [global.LE, fun.Closure]
    function fun1() {
        debugger
        console.log(a)
    }
    fun1()
    let obj = {
        // 6: 遇到函数，解析到函数会使用b，所以 fun.Closure={a:1,b:2}
        // 7: method.[[Scopes]] = [global.LE, fun.Closure]
        method(){
            console.log(b)
        }
    }
}

// 执行到这里时，预编译 fun
fun()
```

> 1、2发生在全局代码的预编译阶段，3、4、5、6、7发生在 fun 的预编译阶段。

> 对于 global.LE，不同环境下的 global.LE 内容不一样，浏览器环境下的作用域链顶层是 [window, Script]，并且 script 作用域不会产生闭包对象。但是 node 环境下是 [global, Script.Closure] , node 环境下 Script 是会产生闭包的。

fun1 执行时的作用域链是这样的：[fun1.LE, fun.Closure, global.LE]

<img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb8e4cc479254597aa777f3d8b57d1ef~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?" alt="image.png" style="zoom: 67%;" />

我们可以看到 `fun1` 的作用域链中的确不存在 `fun.AO` 或者 `fun.LE` ，而是存在 `fun.Closure`。并且 `fun.Closure` 中的内容是 `a` 和 `b` 两个变量，并没有 `c`。这足以证明所有子函数使用的是同一个闭包对象。

细心的你会发现 `Closure` 在 `method` 的定义执行前就已经包含 `b` 变量，这说明 `Closure` 在函数执行前早已确定好了，还有一点就是 `Closure` 中的变量存储的是对应变量的引用地址，如果这个变量值发生变化，那么 `Closure` 中对应的变量也会发生变化(后面会证明)

而且这里 `fun1` 并没有返回到外部调用形成网络上描述的闭包（网络上很多说法是需要返回一个函数才会形成闭包，很显然这也是不对的），而是直接在函数内部同步调用。

结论：每一个函数都会产生闭包，无论 **闭包中是否存在内部函数** 或者 **内部函数中是否访问了当前函数变量** 又或者 **是否返回了内部函数**，因为<u>闭包在当前函数预编译阶段就已经创建了</u>。

### 内存泄漏

说到闭包那么就不得不说内存泄漏，首先我们要搞清楚为什么会内存泄漏？

所谓闭包产生的内存泄漏就是因为闭包对象 `Closure` 无法被释放回收，那么什么情况下 `Closure` 才会被回收呢？

这当然是在<u>没有任何地方引用 `Closure` 的时候</u>，因为 `Closure` 会被所有的子函数的作用域链 `[[Scopes]]` 引用，所以想要 `Closure` 不被引用就需要所有子函数都被销毁，从而导致所有子函数的 `[[Scopes]]` 被销毁，然后 `Closure` 才会被销毁。

这与许多网络上的资料是不一样的，常见的说法是必须返回的函数中使用的自由变量才会产生闭包，也就是下面这样

```js
function fun(){
    let arr = Array(10000000)
    return function(){
        console.log(arr);// 使用了 arr
    }
}
window.f = fun()
```

但是其实不然，即使返回的的函数没有访问自由变量，只要有任何一个函数将 arr 添加到闭包对象 `Closure` 中，arr 都不会正常被销毁，所以下面两段代码都会产生内存泄漏

```js
function fun(){
    let arr = Array(10000000)
    function fun1(){// arr 加入 Closure
        console.log(arr)
    }
    return function fun2(){}
}
window.f = fun()// 全局变量长久持有fun2的引用
```

> 因为 `fun1` 让 `arr` 加入了 `fun.Closure`，`fun2` 又被 `window.f` 持有引用无法释放，因为 `fun2` 的作用域链同样包含 `fun.Closure`，所以 `fun.Closure `也无法释放，最终导致 `arr` 无法释放产生内存泄漏。

```js
function fun(){
    let arr = Array(10000000)
    function fun1() {// arr 加入 Closure
        console.log(arr)
    }
    window.obj = {// 长久持有 window.obj.method 的引用
        method(){}
    }
}
fun()
```

> 同理是因为 `window.obj.method` 作用域链持有 `fun` 的 `Closure` 引用导致 `arr` 无法释放。

那么我们应该如何释放`arr`呢？答案是将 `arr = null` 置为null。

#### 经典的例子

下面是一个经典的内存泄漏的例子，在大多数与闭包内存泄漏的文章或者书籍中都能看到他的影子

```js
let theThing = null;
let replaceThing = function () {
    let leak = theThing;
    function unused () { 
        if (leak){}
    };

    theThing = {  
        longStr: new Array(1000000),
        someMethod: function () {  
                                   
        }
    };
};

let index = 0;
while(index < 100){
    replaceThing()
    index++;
}
```

https://juejin.cn/post/7079995358624874509#heading-4

### 总结

- 每一个函数在执行之前都会进行**预编译**，预编译时会创建一个空的闭包对象`Closure`。
- 每当这个函数预编译时**遇到其内部的函数声明**时，会快速的**扫描**内部函数使用了当前函数中的哪些变量，将可能使用到的变量加入到闭包对象中，最终这个闭包对象将作为这些内部函数作用域链中的一员。
- 只有**所有内部函数的作用域链都被释放**才会释放当前函数的闭包对象，所谓的闭包内存泄漏也就是**因为闭包对象无法释放**产生的。
- 我们还介绍的一个巧妙且经典的内存泄漏案例，并且通过一些demo的运行结果证明了上面这些结论的正确性。

## 原型链

### 1.原型对象和构造函数有何关系？

在JavaScript中，每当定义一个函数数据类型(普通函数、类)时候，都会天生自带一个prototype属性，这个属性指向函数的原型对象。

当函数经过**new调用**时，这个函数就**成为了构造函数**，返回一个全新的实例对象，这个实例对象有一个`__proto__`属性，指向构造函数的原型对象。

![img](\images\16de955a81892535tplv-t2oaga2asx-zoom-in-crop-mark1304000.awebp)

![img](\images\16de955ca89f6091tplv-t2oaga2asx-zoom-in-crop-mark1304000.awebp)

### 2.能不能描述一下原型链？

JavaScript对象通过`__proto__` 指向父类对象，直到指向Object对象为止，这样就形成了一个原型指向的链条, 即原型链。

- 对象的 hasOwnProperty() 来检查对象**自身**中是否含有该属性
- 使用 in 检查对象中是否含有某个属性时，如果**对象中没有但是原型链中有**，也会返回 true

![WechatIMG114.jpeg](\images\d9afcd1172d340508d25c095b1103factplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)

## 继承



## 函数的arguments特点

JavaScript深入之类数组对象与arguments

https://github.com/mqyqingfeng/Blog/issues/14



### 类数组对象

所谓的类数组对象:

> 拥有一个 length 属性和若干索引属性的对象

举个例子：

```js
var array = ['name', 'age', 'sex'];

var arrayLike = {
    0: 'name',
    1: 'age',
    2: 'sex',
    length: 3
}
```

即便如此，为什么叫做类数组对象呢？

那让我们从读写、获取长度、遍历三个方面看看这两个对象。

#### 读写

```js
console.log(array[0]); // name
console.log(arrayLike[0]); // name

array[0] = 'new name';
arrayLike[0] = 'new name';
```

#### 长度

```js
console.log(array.length); // 3
console.log(arrayLike.length); // 3
```

#### 遍历

```js
for(var i = 0, len = array.length; i < len; i++) {
   ……
}
for(var i = 0, len = arrayLike.length; i < len; i++) {
    ……
}
```

是不是很像？

那类数组对象可以使用数组的方法吗？比如：

```js
arrayLike.push('4');
```

然而上述代码会报错: arrayLike.push is not a function

所以终归还是类数组呐……

#### 调用数组方法

如果类数组就是任性的想用数组的方法怎么办呢？

既然无法直接调用，我们可以用 Function.call 间接调用：

```js
var arrayLike = {0: 'name', 1: 'age', 2: 'sex', length: 3 }

Array.prototype.join.call(arrayLike, '&'); // name&age&sex

Array.prototype.slice.call(arrayLike, 0); // ["name", "age", "sex"] 
// slice可以做到类数组转数组

Array.prototype.map.call(arrayLike, function(item){
    return item.toUpperCase();
}); 
// ["NAME", "AGE", "SEX"]
```

#### 类数组转数组

在上面的例子中已经提到了一种类数组转数组的方法，再补充三个：

```js
var arrayLike = {0: 'name', 1: 'age', 2: 'sex', length: 3 }
// 1. slice
Array.prototype.slice.call(arrayLike); // ["name", "age", "sex"] 
// 2. splice
Array.prototype.splice.call(arrayLike, 0); // ["name", "age", "sex"] 
// 3. ES6 Array.from
Array.from(arrayLike); // ["name", "age", "sex"] 
// 4. apply
Array.prototype.concat.apply([], arrayLike)
```

那么为什么会讲到类数组对象呢？以及类数组有什么应用吗？

要说到类数组对象，Arguments 对象就是一个类数组对象。在客户端 JavaScript 中，一些 DOM 方法(document.getElementsByTagName()等)也返回类数组对象。

### Arguments对象

接下来重点讲讲 Arguments 对象。

Arguments 对象只定义在函数体中，包括了函数的参数和其他属性。在函数体中，arguments 指代该函数的 Arguments 对象。

举个例子：

```js
function foo(name, age, sex) {
    console.log(arguments);
}

foo('name', 'age', 'sex')
```

打印结果如下：

[![arguments](\images\68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6d717971696e6766656e672f426c6f672f496d616765732f617267756d656e74732e706e67.png)](https://camo.githubusercontent.com/993a101381ec9e9badf6591d841fd7deb53a7a8dde01bf17980cc2aefacc65d4/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f6d717971696e6766656e672f426c6f672f496d616765732f617267756d656e74732e706e67)

我们可以看到除了类数组的索引属性和length属性之外，还有一个callee属性，接下来我们一个一个介绍。

#### length属性

Arguments对象的length属性，表示实参的长度，举个例子：

```js
function foo(b, c, d){
    console.log("实参的长度为：" + arguments.length)
}

console.log("形参的长度为：" + foo.length)

foo(1)

// 形参的长度为：3
// 实参的长度为：1
```

#### callee属性

Arguments 对象的 callee 属性，通过它可以**调用函数自身**。

讲个闭包经典面试题使用 callee 的解决方法：

```js
var data = [];

for (var i = 0; i < 3; i++) {
    (data[i] = function () {
       console.log(arguments.callee.i) 
    }).i = i;
}

data[0]();
data[1]();
data[2]();

// 0
// 1
// 2
```

接下来讲讲 arguments 对象的几个注意要点：

#### arguments 和对应参数的绑定

```js
function foo(name, age, sex, hobbit) {

    console.log(name, arguments[0]); // name name

    // 改变形参
    name = 'new name';

    console.log(name, arguments[0]); // new name new name

    // 改变arguments
    arguments[1] = 'new age';

    console.log(age, arguments[1]); // new age new age

    // 测试未传入的是否会绑定
    console.log(sex); // undefined

    sex = 'new sex';

    console.log(sex, arguments[2]); // new sex undefined

    arguments[3] = 'new hobbit';

    console.log(hobbit, arguments[3]); // undefined new hobbit

}

foo('name', 'age')
```

**传入的参数，实参和 arguments 的值会共享**，**当没有传入时，实参与 arguments 值不会共享**

除此之外，以上是在非严格模式下，如果是在严格模式下，实参和 arguments 是不会共享的。

#### 传递参数

将参数从一个函数传递到另一个函数

```js
// 使用 apply 将 foo 的参数传递给 bar
function foo() {
    bar.apply(this, arguments);
}
function bar(a, b, c) {
   console.log(a, b, c);
}

foo(1, 2, 3)
```

#### 强大的ES6

使用ES6的 ... 运算符，我们可以轻松转成数组。

```js
function func(...arguments) {
    console.log(arguments); // [1, 2, 3]
}

func(1, 2, 3);
```

### 函数的arguments为什么不是数组？如何转化成数组？

因为arguments本身并不能调用数组方法，它是一个另外一种对象类型，只不过属性从0开始排，依次为0，1，2...最后还有callee和length属性。我们也把这样的对象称为**类数组**。

常见的类数组还有：

- 1. 用getElementsByTagName/ClassName()获得的HTMLCollection
- 1. 用querySelector获得的nodeList

那这导致很多数组的方法就不能用了，必要时需要我们将它们转换成数组，有哪些方法呢？


```js
var arrayLike = {0: 'name', 1: 'age', 2: 'sex', length: 3 }
// 1. slice
Array.prototype.slice.call(arrayLike); // ["name", "age", "sex"] 
// 2. splice
Array.prototype.splice.call(arrayLike, 0); // ["name", "age", "sex"] 
// 3. ES6 Array.from
Array.from(arrayLike); // ["name", "age", "sex"] 
// 4. apply
Array.prototype.concat.apply([], arrayLike)
```

## 内存

网上的资料基本是这样说的: 基本数据类型用`栈`存储，引用数据类型用`堆`存储。

看起来没有错误，但实际上是有问题的。可以考虑一下闭包的情况，如果变量存在栈中，那函数调用完`栈顶空间销毁`，闭包变量不就没了吗？

其实还是需要补充一句:

> 闭包变量是存在堆内存中的。

### 内存分类

JS内存空间分为**栈（stack）内存和堆（heap）内存**，栈内存是栈结构存储**基本数据类型**和**指向堆内存的指针**，堆内存**存储复杂数据类型**。

![img](\images\170ce92a2714913etplv-t2oaga2asx-zoom-in-crop-mark3024000.awebp)

值得注意的是，对于`赋值`操作，原始类型的数据直接完整地复制变量值，对象数据类型的数据则是复制引用地址。

因此会有下面的情况:

```js
let obj = { a: 1 };
let newObj = obj;
newObj.a = 2;
console.log(obj.a);//变成了2
```

之所以会这样，是因为 obj 和 newObj 是同一份堆空间的地址，改变newObj，等于改变了共同的堆内存，这时候通过 obj 来获取这块内存的值当然会改变。

### 变量声明与赋值

#### 核心点总结

- 变量声明的本质是变量名与栈内存地址进行绑定，不直接与堆内存进行绑定。
- 声明的基本数据类型会将值存储在栈内存中，声明的复杂数据类型会将值存储在堆内存中并将其在堆中的内存地址作为值存到栈内存中。
- const声明常量本质是指的是声明的变量名所指向的栈内存地址不可改变，但是栈中对应的值可以改变。
- 基本数据类型赋值是在栈内存中申请新的内存区域保存值并将其指向的内存地址绑定到原有变量上。
- 复杂数据类型赋值是在堆内存中申请新的内存区域保存值并将其指向的内存地址作为值在栈内存中申请新的内存区域保存将其在栈中的内存地址绑定到变量上。

#### 过程详解

https://juejin.cn/post/6844904088304353293#heading-6

### 为什么不用栈保存复杂类型数据

当然，你可能会问: 为什么不全部用栈来保存呢？

首先，对于系统栈来说，它的功能除了保存变量之外，还有**创建并切换函数执行上下文的功能**。举个例子:

```js
function f(a) {
  console.log(a);
}

function func(a) {
  f(a);
}

func(1);
```

假设用ESP指针来保存当前的执行状态，在系统栈中会产生如下的过程：

1. 调用func, 将 func 函数的上下文压栈，ESP指向栈顶。
2. 执行func，又调用f函数，将 f 函数的上下文压栈，ESP 指针上移。
3. 执行完 f 函数，将ESP 下移，f函数对应的栈顶空间被回收。
4. 执行完 func，ESP 下移，func对应的空间被回收。

图示如下:

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/23/16e96b6b57b734c1~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)

因此你也看到了，如果采用栈来存储相对基本类型更加复杂的对象数据，那么切换上下文的开销将变得巨大！

不过堆内存虽然空间大，能存放大量的数据，但与此同时垃圾内存的回收会带来更大的开销





## 内存泄漏

### 什么是内存泄漏

本质上讲,内存泄漏就是由于疏忽或错误造成程序未能释放那些已经不再使用的内存，造成内存的浪费。

### 内存泄漏的检测

**在 Chrome 浏览器中，我们可以这样查看内存占用情况**

1. 打开开发者工具，选择 Performance 面板
2. 在顶部勾选 Memory
3. 点击左上角的 record 按钮
4. 在页面上进行各种操作，模拟用户的使用情况
5. 一段时间后，点击对话框的 stop 按钮，面板上就会显示这段时间的内存占用情况

来看一张效果图：



![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/6/17/16b6373703dd6747~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)



我们有两种方式来判定当前是否有内存泄漏：

1. 多次快照后，比较每次快照中内存的占用情况，如果呈上升趋势，那么可以认为存在内存泄漏
2. 某次快照后，看当前内存占用的趋势图，如果走势不平稳，呈上升趋势，那么可以认为存在内存泄漏



### 内存泄漏的场景

#### 意外的全局变量

```js
function foo() {
    bar1 = 'some text'; // 没有声明变量 实际上是全局变量 => window.bar1
    this.bar2 = 'some text' // 全局变量 => window.bar2
}
foo();
```

在这个例子中，意外的创建了两个全局变量 bar1 和 bar2

#### 被遗忘的定时器和回调函数

在很多库中, 如果使用了观察者模式, 都会提供回调方法, 来调用一些回调函数。 要记得回收这些回调函数。举一个 setInterval的例子：

```js
var serverData = loadData();
setInterval(function() {
    var renderer = document.getElementById('renderer');
    if(renderer) {
        renderer.innerHTML = JSON.stringify(serverData);
    }
}, 5000); // 每 5 秒调用一次
```

如果后续 renderer 元素被移除，整个定时器实际上没有任何作用。 但如果你没有回收定时器，整个定时器依然有效, **不但定时器无法被内存回收**， **定时器函数中的依赖也无法回收**。在这个案例中的 serverData 也无法被回收。

#### 闭包

在 JS 开发中，我们会经常用到闭包，一个内部函数，有权访问包含其的外部函数中的变量。 下面这种情况下，闭包也会造成内存泄露:

```js
var theThing = null;
var replaceThing = function () {
  var originalThing = theThing;
  var unused = function () {
    if (originalThing) // 对于 'originalThing'的引用
      console.log("hi");
  };
  theThing = {
    longStr: new Array(1000000).join('*'),
    someMethod: function () {
      console.log("message");
    }
  };
};
setInterval(replaceThing, 1000);
```

这段代码，每次调用 replaceThing 时，theThing 获得了包含一个巨大的数组和一个对于新闭包 someMethod 的对象。 同时 unused 是一个引用了 originalThing 的闭包。

这个范例的关键在于，闭包之间是共享作用域的，尽管 unused 可能一直没有被调用，但是 someMethod 可能会被调用，就会导致无法对其内存进行回收。 当这段代码被反复执行时，内存会持续增长。

#### DOM 引用

很多时候, 我们对 Dom 的操作, 会把 Dom 的引用保存在一个数组或者 Map 中。

```js
var elements = {
    image: document.getElementById('image')
};
function doStuff() {
    elements.image.src = 'http://example.com/image_name.png';
}
function removeImage() {
    document.body.removeChild(document.getElementById('image'));
    // 这个时候我们对于 #image 仍然有一个引用, Image 元素, 仍然无法被内存回收.
}
```

上述案例中，**即使我们对于 image 元素进行了移除，但是仍然有对 image 元素的引用**，依然无法对其进行内存回收。

另外需要注意的一个点是，对于一个 Dom 树的叶子节点的引用。 举个例子: 如果我们引用了一个表格中的td元素，一旦在 Dom 中删除了整个表格，我们直观的觉得内存回收应该回收除了被引用的 td 外的其他元素。 但是事实上，这个 td 元素是整个表格的一个子元素，并保留对于其父元素的引用。 这就**会导致对于整个表格，都无法进行内存回收**。所以我们要小心处理对于 Dom 元素的引用。



## V8垃圾回收 GC

有内存就必然有**垃圾回收**（**GC**），JS中栈内存多数是在函数执行时使用（根据函数调用顺序也叫做**调用栈**），函数执行完后即开始栈内存的垃圾回收。堆内存由于存在多个栈内存中的指针指向它以及堆内存较大等原因，需要采用特定的垃圾回收算法处理。

> 垃圾回收的关键在于如何判断内存已经不再使用然后将其释放掉

### v8内存限制

在其他的后端语言中，如Java/Go, 对于内存的使用没有什么限制，但是JS不一样，V8只能使用系统的一部分内存，具体来说，在`64`位系统下，V8最多只能分配`1.4G`, 在 32 位系统中，最多只能分配`0.7G`。你想想在前端这样的大内存需求其实并不大，但对于后端而言，nodejs如果遇到一个2G多的文件，那么将无法全部将其读入内存进行各种操作了。

### 引用计数算法

主要是IE等旧浏览器在采用，通过计数器分析变量的引用次数，清除没有引用到的变量。对于**存在循环引用的情况则无法处理**，比如：

```
function cycle() {
    var o1 = {}
    var o2 = {}
    o1.a = o2
    o2.a = o1
    return "Cycle reference!"
}
cycle()
```

其中 o1 引用了 o2，o2 引用了 o1，在cycle函数执行完 o1，o2 都没有再次引用到，但是引用计数算法判断两者都存在引用。

### 分代式垃圾回收

V8 把堆内存分成了两部分进行处理——新生代内存和老生代内存。

顾名思义，新生代就是临时分配的内存，存活时间短， 老生代是常驻内存，存活的时间长。V8 的堆内存，也就是两个内存之和。

![img](\images\16e96b6ec3859a65tplv-t2oaga2asx-zoom-in-crop-mark3024000.awebp)

根据这两种不同种类的堆内存，V8 采用了不同的回收策略，来根据不同的场景做针对性的优化。

### 新生代内存回收（Scavenge/Cheney）

首先将新生代内存空间一分为二:

![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/11/23/16e96b71923adacb~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp)

其中From部分表示**正在使用**的内存，To 是目前**闲置**的内存。

当进行垃圾回收时，V8 将From部分的对象检查一遍，如果是存活对象那么复制到To内存中(在To内存中按照顺序从头放置的)，如果是非存活对象直接回收即可。

当所有的From中的存活对象按照顺序进入到To内存之后，From 和 To 两者的角色`对调`，From现在被闲置，To为正在使用，如此循环。

详细步骤如下：

1. 新加入的对象都会存放到使用区，当使用区快被写满时，就需要执行一次垃圾清理操作 (GC时机：使用区写满时)

2. 当开始进行垃圾回收时，新生代垃圾回收器会对使用区中的活动对象做标记，标记完成之后将使用区的活动对象复制进空闲区并进行排序

3. 随后进入垃圾清理阶段，即将非活动对象占用的空间清理掉

4. 最后进行角色互换，把原来的使用区变成空闲区，把原来的空闲区变成使用区

#### 新生代转为老生代的两种情况:

（**1. 存活时间长**）当一个对象经过多次复制后依然存活，它将会被认为是生命周期较长的对象，随后会被移动到老生代中，采用老生代的垃圾回收策略进行管理

（**2. 占用内存大）**复制一个对象到空闲区时，空闲区空间占用超过了 25%，那么这个对象会被直接晋升到老生代空间中

#### 垃圾回收器是怎么知道哪些对象是活动对象和非活动对象的

有一个概念叫**对象的可达性**，表示从初始的根对象（window，global）的指针开始，这个根指针对象被称为根集（root set），从这个根集向下搜索其子节点，被搜索到的子节点说明该节点的引用对象可达，并为其留下标记，然后**递归**这个搜索的过程，直到所有子节点都被遍历结束，那么没有被标记的对象节点，说明该对象没有被任何地方引用，可以证明这是一个需要被释放内存的对象，可以被垃圾回收器回收。

#### Scanvenge算法如何解决内存碎片的问题

假设这样的场景:

![img](\images\16e96b73ac9e01cctplv-t2oaga2asx-zoom-in-crop-mark3024000.awebp)

深色的小方块代表存活对象，白色部分表示待分配的内存，由于堆内存是连续分配的，这样零零散散的空间可能会导致稍微大一点的对象没有办法进行空间分配，这种零散的空间也叫做**内存碎片**。

Scanvenge算法每次回收将From部分的存活对象复制到To内存，按照顺序从头放置。

在复制之后，To空间变成了这个样子:

![img](\images\16e96b7741afdb10tplv-t2oaga2asx-zoom-in-crop-mark3024000-16558685891546.awebp)

这样就解决了内存碎片的问题，大大方便了后续连续空间的分配。

#### Scanvenge算法的特点

**劣势**：

内存只能使用新生代内存的一半

**优势**：

由于它只存放生命周期短的对象，这种对象`一般很少`，因此`时间`性能非常优秀。

### 老生代内存的回收（Mark-Sweep & Mark-Compact）

刚刚介绍了新生代的回收方式，那么新生代中的变量如果经过多次回收后依然存在，那么就会被放入到`老生代内存`中，这种现象就叫`晋升`。

#### 新生代转为老生代的两种情况:

**1. 存活时间长**：当一个对象经过多次复制后依然存活，它将会被认为是生命周期较长的对象，随后会被移动到老生代中，采用老生代的垃圾回收策略进行管理

**2. 占用内存大**：复制一个对象到空闲区时，空闲区空间占用超过了 25%，那么这个对象会被直接晋升到老生代空间中

#### 为什么老生代不使用scanvenge算法

- scavenge为**复制**算法，重复复制活动对象会使得效率低下
- scavenge是牺牲**空间来换取时间**效率的算法，而老生代支持的容量较大，会出现**空间资源浪费**问题。

#### 标记整理

第一步，进行标记-清除。这个过程在《JavaScript高级程序设计(第三版)》中有过详细的介绍，主要分成两个阶段，即**标记阶段**和**清除阶段**。

首先会遍历堆中的所有对象，对它们做上标记，然后对于代码环境中`使用的变量`以及被`强引用`的变量取消标记，剩下的就是要删除的变量了，在随后的`清除阶段`对其进行空间的回收。

当然这又会引发内存碎片的问题，存活对象的空间不连续对后续的空间分配造成障碍。老生代又是如何处理这个问题的呢？

第二步，整理内存碎片。V8 的解决方式非常简单粗暴，在清除阶段结束后，把存活的对象全部往一端靠拢。

![img](\images\16e96b7a41c8c826tplv-t2oaga2asx-zoom-in-crop-mark3024000.awebp)

由于是移动对象，它的执行速度不可能很快，事实上也是整个过程中最耗时间的部分。

### V8对GC的其余优化

#### 全停顿的概念 Stop-the-World

由于垃圾回收是在JS引擎中进行的，而Mark-Compact算法在执行过程中需要移动对象，而当活动对象较多的时候，它的执行速度不可能很快，为了避免JavaScript应用逻辑和垃圾回收器的内存资源竞争导致的不一致性问题，垃圾回收器会将JavaScript应用暂停，这个过程，被称为`全停顿`（stop-the-world）。

在新生代中，由于空间小、存活对象较少、Scavenge算法执行效率较快，所以全停顿的影响并不大。而老生代中就不一样，如果老生代中的活动对象较多，垃圾回收器就会暂停主线程较长的时间，使得页面变得卡顿。

#### 增量标记 - Incremental marking

为了降低全堆垃圾回收的停顿时间，增量标记将原本的标记全堆对象拆分为一个一个任务，让其**穿插在JavaScript应用逻辑之间**执行，它允许堆的标记时的5~10ms的停顿。增量标记在堆的大小达到一定的阈值时启用，启用之后每当一定量的内存分配后，脚本的执行就会停顿并进行一次增量标记。

![img](\images\1460000025129644.png)

#### 懒性清理 - Lazy sweeping

增量标记只是对活动对象和非活动对象进行标记，惰性清理用来真正的清理释放内存。当增量标记完成后，假如当前的可用内存足以让我们快速的执行代码，其实我们是没必要立即清理内存的，可以将清理的过程延迟一下，让JavaScript逻辑代码先执行，也无需一次性清理完所有非活动对象内存，垃圾回收器会按需逐一进行清理，直到所有的页都清理完毕。

增量标记与惰性清理的出现，使得主线程的最大停顿时间减少了80%，让用户与浏览器交互过程变得流畅了许多，从实现机制上，由于每个小的增量标价之间执行了JavaScript代码，堆中的对象指针可能发生了变化，需要使用`写屏障`技术来记录这些引用关系的变化，所以也暴露出来增量标记的缺点：

- 并没有减少主线程的总暂停的时间，甚至会略微增加
- 由于写屏障（Write-barrier）机制的成本，增量标记可能会降低应用程序的吞吐量

#### 并发 - Concurrent

并发式GC允许在**在垃圾回收的同时不需要将主线程挂起**，两者可以同时进行，只有在个别时候需要短暂停下来让垃圾回收器做一些特殊的操作。但是这种方式也要面对增量回收的问题，就是在垃圾回收过程中，由于JavaScript代码在执行，堆中的对象的引用关系随时可能会变化，所以也要进行`写屏障`操作。

![img](\images\1460000025129641.png)

#### 并行 - Parallel

并行式GC允许主线程和辅助线程同时执行同样的GC工作，这样可以让辅助线程来分担主线程的GC工作，使得垃圾回收所耗费的时间等于**总时间除以参与的线程数量**（加上一些同步开销）。

![img](\images\1460000025129642.png)

### V8当前GC的机制

2011年，V8应用了增量标记机制。直至2018年，Chrome64和Node.js V10启动并发标记（Concurrent），同时在并发的基础上添加并行（Parallel）技术，使得垃圾回收时间大幅度缩短。

#### 新生代：并行回收Parellel

V8在新生代垃圾回收中，使用并行（parallel）机制，在整理排序阶段，也就是将活动对象从`from-to`复制到`space-to`的时候，**启用多个辅助线程**，并行的进行整理。由于多个线程竞争一个新生代的堆的内存资源，可能出现有某个活动对象被多个线程进行复制操作的问题，为了解决这个问题，V8在第一个线程对活动对象进行复制并且复制完成后，都必须去维护复制这个活动对象后的指针转发地址，以便于其他协助线程可以找到该活动对象后可以判断该活动对象是否已被复制。

![img](\images\1460000025129645.png)

#### 老生代垃圾回收：并发Concurrent

V8在老生代垃圾回收中，如果堆中的内存大小超过某个阈值之后，会启用并发（Concurrent）标记任务。每个辅助线程都会去追踪每个标记到的对象的指针以及对这个对象的引用，而在JavaScript代码执行时候，并发标记也在后台的辅助进程中进行，当堆中的某个对象指针被JavaScript代码修改的时候，写入屏障（[write barriers](https://link.segmentfault.com/?enc=p529SsNxTEFN8MtRC9CyhA%3D%3D.oUSQcAIcDBGi4Bc5A4b7LB3W7zCDK%2FTu5Dv4y8ydOeiKTy8NqQRNUa8KJfcTLUj09ozxcw5XURpt3G9tcqhc8GtmnilK%2BCJ1SNJrNoYpLOk%3D)）技术会在辅助线程在进行并发标记的时候进行追踪。

当并发标记完成或者动态分配的内存到达极限的时候，主线程会**执行最终的快速标记步骤**，这个时候**主线程会挂起**，主线程会再一次的扫描根集以确保所有的对象都完成了标记，由于辅助线程已经标记过活动对象，主线程的本次扫描只是进行check操作，确认完成之后，某些辅助线程会进行清理内存操作，某些辅助进程会进行内存整理操作，由于都是并发的，并不会影响主线程JavaScript代码的执行。

![img](\images\1460000025129643.png)

#### 参考

https://segmentfault.com/a/1190000025129635

https://juejin.cn/post/6844904004007247880#heading-1

## V8执行一段JS代码的过程

首先需要明白的是，机器是读不懂 JS 代码，机器只能理解特定的机器码，那如果要让 JS 的逻辑在机器上运行起来，就必须将 JS 的代码翻译成机器码，然后让机器识别。JS属于解释型语言，对于解释型的语言说，解释器会对源代码做如下分析:

- 通过词法分析和语法分析生成 AST(抽象语法树)
- 生成字节码

然后解释器根据字节码来执行程序。但 JS 整个执行的过程其实会比这个更加复杂，接下来就来一一地拆解。

### 1.生成 AST

生成 AST 分为两步——词法分析和语法分析。

词法分析即分词，它的工作就是将一行行的代码分解成一个个token。 比如下面一行代码:

```js
let name = 'sanyuan'
```

其中会把句子分解成四个部分:

![img](\images\16e96b7d3513ebf5tplv-t2oaga2asx-zoom-in-crop-mark3024000.awebp)

即解析成了四个token，这就是词法分析的作用。

接下来语法分析阶段，将生成的这些 token 数据，根据一定的语法规则转化为AST。举个例子:

```js
let name = 'sanyuan'
console.log(name)
```

最后生成的 AST 是这样的:

![img](\images\16e96b7ff6b0f513tplv-t2oaga2asx-zoom-in-crop-mark3024000.awebp)

当生成了 AST 之后，编译器/解释器后续的工作都要依靠 AST 而不是源代码。

### 2. 生成字节码

开头就已经提到过了，生成 AST 之后，直接通过 V8 的解释器(也叫Ignition)来生成字节码。但是`字节码`并不能让机器直接运行，那你可能就会说了，不能执行还转成字节码干嘛，直接把 AST 转换成机器码不就得了，让机器直接执行。确实，在 V8 的早期是这么做的，但后来因为机器码的体积太大，引发了严重的内存占用问题。

给一张对比图让大家直观地感受以下三者代码量的差异:

![img](\images\16e96b822da9857ctplv-t2oaga2asx-zoom-in-crop-mark3024000.awebp)

很容易得出，字节码是比机器码轻量得多的代码。那 V8 为什么要使用字节码，字节码到底是个什么东西？

> 字节码是介于AST 和 机器码之间的一种代码，但是与特定类型的机器码无关，字节码需要通过解释器将其转换为机器码然后执行。

字节码仍然需要转换为机器码，但和原来不同的是，现在不用一次性将全部的字节码都转换成机器码，而是通过解释器来逐行执行字节码，省去了生成二进制文件的操作，这样就大大降低了内存的压力。

### 3. 执行代码

接下来，就进入到字节码解释执行的阶段啦！

在执行字节码的过程中，如果发现某一部分代码重复出现，那么 V8 将它记做`热点代码`(HotSpot)，然后将这么代码编译成`机器码`保存起来，这个用来编译的工具就是V8的`编译器`(也叫做`TurboFan`) , 因此在这样的机制下，代码执行的时间越久，那么执行效率会越来越高，因为有越来越多的字节码被标记为`热点代码`，遇到它们时直接执行相应的机器码，不用再次将转换为机器码。

其实当你听到有人说 JS 就是一门解释器语言的时候，其实这个说法是有问题的。因为字节码不仅配合了解释器，而且还和编译器打交道，所以 JS 并不是完全的解释型语言。而编译器和解释器的 根本区别在于前者会编译生成二进制文件但后者不会。

并且，这种字节码跟编译器和解释器结合的技术，我们称之为`即时编译`, 也就是我们经常听到的`JIT`。

这就是 V8 中执行一段JS代码的整个过程，梳理一下:

1. 首先通过词法分析和语法分析生成 `AST`
2. 将 AST 转换为字节码
3. 由解释器逐行执行字节码，遇到热点代码启动编译器进行编译，生成对应的机器码, 以优化执行效率

## 理解EventLoop

### 浏览器中的EventLoop

#### 

## 理解EventLoop- Node.js

### Node中的EventLoop与浏览器的差异

https://nodejs.org/zh-cn/docs/guides/event-loop-timers-and-nexttick/

nodejs 和 浏览器的 eventLoop 还是有很大差别的.

Node.js 是一个新的 JS 运行环境，它同样要支持异步逻辑，包括定时器、IO、网络请求，很明显，也可以用 Event Loop 那一套来跑。

但是呢，浏览器那套 Event Loop 就是为浏览器设计的，对于做高性能服务器来说，那种设计还是有点粗糙了。

浏览器的 Event Loop 只分了两层优先级，一层是宏任务，一层是微任务。但是**宏任务之间没有再划分优先级**，**微任务之间也没有再划分优先级**。

而 Node.js 任务**宏任务之间也是有优先级**的，<u>比如定时器 Timer 的逻辑就比 IO 的逻辑优先级高，因为涉及到时间，越早越准确；而 close 资源的处理逻辑优先级就很低，因为不 close 最多多占点内存等资源，影响不大</u>。

于是就把宏任务队列拆成了五个优先级：Timers、Pending、Poll、Check、Close。

![img](\images\16e96b8587ad911dtplv-t2oaga2asx-zoom-in-crop-mark3024000.awebp)

![img](\images\2f16ec03bf614d5b9d01fe55b126758btplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp)

### 五大阶段

解释一下这五个阶段，每一层你可以将它理解成为一个队列（也就是说宏任务存在这几种不同优先级的队列）:

1. **Timers Callback**： 涉及到时间，肯定越早执行越准确，所以这个优先级最高很容易理解。
2. **Pending Callback**：上一次循环队列中，还未执行完毕的会在这个阶段进行执行。比如处理网络、IO 等异常时的回调。
3. **Poll Callback**：处理新的 I/O 相关的回调, 例如I/O 的 data，网络的 connection。需要注意的是这一阶段会存在阻塞
4. **Check Callback**：执行 `setImmediate` 的回调，特点是刚执行完 I/O 之后就能回调这个。
5. **Close Callback**：关闭资源的回调，晚点执行影响也不大，所以优先级最低。比如：`socket.on('close', ...)`

#### 1. 三大关键阶段

首先，梳理一下 nodejs 三个非常重要的执行阶段:

1. `timer` 的阶段。检查定时器，如果到了时间，就执行回调。这些定时器就是setTimeout、setInterval。这个阶段暂且叫它`timer`。每当执行完毕同时仍然会进行 Prcess.nextTick -> micro 的步骤从而清空下一个 timer 任务。

2. `poll`阶段。因为在node代码中难免会有异步操作，比如文件I/O，网络I/O等等，那么当这些异步操作做完了，就会来通知JS主线程，怎么通知呢？就是通过'data'、'connect'等事件使得事件循环到达 `poll` 阶段。到达了这个阶段后: 如果当前已经存在定时器，而且有定时器到时间了，拿出来执行，eventLoop 将回到timer阶段。如果没有定时器, 会去看回调函数队列。

   - 如果队列`不为空`，拿出队列中的方法依次执行

   - 如果队列为空，检查是否有 `setImmdiate` 的回调
     - 有则前往`check阶段`(下面会说)
     - `没有则继续等待`，相当于阻塞了一段时间(阻塞时间是有上限的), 等待 callback 函数加入队列，加入后会立刻执行。一段时间后`自动进入 check 阶段`。

3. `check` 阶段。这是一个比较简单的阶段，直接执行` setImmdiate` 的回调。

这三个阶段为一个循环过程。不过现在的eventLoop并不完整，我们现在就来一一地完善。

#### 2. 完善

首先，当第 1 阶段结束后，可能并不会立即等待到异步事件的响应，这时候 nodejs 会进入到 `I/O异常的回调阶段`。比如说 TCP 连接遇到ECONNREFUSED，就会在这个时候执行回调。

并且在 check 阶段结束后还会进入到 `关闭事件的回调阶段`。如果一个 socket 或句柄（handle）被突然关闭，例如 socket.destroy()， 'close' 事件的回调就会在这个阶段执行。

梳理一下，nodejs 的 eventLoop 分为下面的几个阶段:

1. timer 阶段
2. I/O 异常回调阶段
3. idle prepare状态(第2阶段结束，poll 未触发之前)
4. poll 阶段
5. check 阶段
6. close阶段

### EventLoop一些值得注意的问题

#### `setTimeout` & `setImmediate` 谁快？

```js
function timer() {
  console.log('timer');
}

function immediate() {
  console.log('immediate');
}

setTimeout(() => {
  timer();
}, 0);

setImmediate(() => {
  immediate();
});
```

我们先来尝试分析这段代码。首先，我们说过当脚本执行完毕时（我们可以理解为同步代码执行完毕时），这段代码会在 timer 以及 check 阶段的队列中分别推入对应的 timer 函数和 immediate 函数。

按照我们的理解，当同步脚本执行完毕后：

- 首先会检查是否存在 process.nextTick ，显示代码中是不存在任何 nextTick 相关调用。所以会跳过它。
- 其次会进入所谓的 EventLoop 也就是 timer 阶段，因为我们代码中存在定时器函数 setTimerout(timer,0)。

所以到 Loop 到 timer 阶段时，因为定时器满足时间它应该被推入对应的 timers 队列中。当 EventLoop 执行到 timer 阶段时，会拿出这个 timer 的 callback 执行它。

所以不难想象，控制台会执行这个函数打印 timer 。

- 接下来会依次进入 pending callbacks ，显示上一次 EventLoop 中并不存在任何达到上限的操作。所以它是空的。
- 依次进入 idle prepare 阶段。
- **注意，此时我们会进入 poll 轮询阶段，此时 poll 阶段并不存在任何 IO 相关回调，返回在轮询阶段他会检测到我们代码中存在 setImmediate ，并且 setImmediate 的 callback 也已经被推入到了 check 阶段。**
- 所以，在 poll 并不会产生所谓的阻塞效果。会进入 check 阶段，调用代码中 setImmediate 产生的回调函数 immediate ，所以控制台会输出 immediate 。
- 其次，check 阶段清完成 immediate 后，会进入 Loop 中最后的 close callbacks 中。

**显示，根据我们的分析一切都显得那么美好，控制台应该先打印 timer ，其次打印 immediate 。**

但事实上先timer还是先immedate都是有可能的。

问题的本质在于Node.js中的`setTimeout(cb,0)`实际上存在最小执行时间 1 ms，它是会被当作 setTimeout(cb,1) 来执行。

> When `delay` is larger than `2147483647` or less than `1`, the `delay` will be set to `1`. Non-integer delays are truncated to an integer.

恰恰是因为 setTimeout 存在 1ms 的最小间接，如果我们的电脑性能足够好的话。

**那么在上述的同步代码执行完毕，以及进入 EventLoop 中这一切发生在 1ms 之内，显然 timers 阶段由于代码中的 setTimeout 并没有达到对应的时间，换句话说它所对应的 callback 并没有被推入当前 timer 中。**

自然，名为 timer 的函数也并不会被执行。会依次进入接下里的阶段。Loop 会依次向下进行检查。

**当执行到 poll 阶段时，即使定时器对应的 timer 函数已经被推入 timers 中了。由于 poll 阶段检查到存在 setImmediate 所以会继续进入 check 阶段并不会掉头重新进入 timers 中。**

#### Node EventLoop中的微任务

```js
setImmediate(() => {
  console.log('immediate 开始')
  Promise.resolve().then(console.log('immediate' + 1));
  Promise.resolve().then(console.log('immediate' + 2));
  console.log('immediate 结束');
});

setTimeout(() => {
  console.log('timer 开始');
  Promise.resolve().then(console.log('timer' + 1));
  Promise.resolve().then(console.log('timer' + 2));
  console.log('timer 结束');
}, 0);

/*  log
    immediate1 开始
    immediate1 结束
    immediate1 微任务执行
    immediate2 微任务执行
    immediate2 开始
    immediate2 结束
    immediate3 微任务执行
    immediate4 微任务执行 */
```

上述的代码关于 Immediate 和 Timeout 究竟我们在之前讲过它们的原因，它并不是我们在这里讨论的重点。

重点是你可以清楚的看到，无论是 immediate 还是 timer 先执行，都会伴随着本次宏任务中产生的 Micro （微任务）一同执行完毕才会进入下一个 Macro 。

其实它的本质和浏览器中是类似的，虽然 NodeJs 下存在多个执行队列，但是每次执行逻辑是相同的：**同样是执行完成一个宏任务后会立即清空当前队列中产生的所有微任务。**

> 注意：在 NodeJs < 10.0 下的版本，它是会清空一整个队列之后才会清空当前队列下的所有 Micro。



其实 NodeJs 中的事件循环机制主要就是基于以上几个阶段，但是对于我们比较重要的来说仅仅只有 `timers`、`poll` 和 `check` 阶段，因为这三个阶段影响着我们代码书写的执行顺序。

至于 pending callbacks、idle, prepare 、close callbacks 其实对于我们代码的执行顺序并不存在什么强耦合，甚至有些时候我们完全不必在意他们。



### Node API

#### `Process.nextTick()`

执行时机即是在同步任务执行完毕后，在所有微任务队列执行之前执行。即将将 micro-task 推入栈中时优先会将 `Process.nextTick` 推入栈中进行执行。

可以认为`Process.nextTick`就是更高优先级的微任务，在所有的普通微任务之前进行（虽然官方并不将它认为是 EventLoop 中的一部分）。

<img src="\images\0b8edd1a735f4323a8400f3e544de7aetplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp" alt="img" style="zoom:50%;" />

#### `setImmediate`

setImmediate 同样是 NodeJs 中的 API。它意为当要异步地（但要尽可能快）执行某些代码时，使用 `setImmediate()` 函数。

它类似于 `setTimeout(()=> {},0)`，所谓 setImmediate 同样是一个 macro 宏任务。关于它和 setTimeout 的执行时机我们会在稍后详细讨论。

#### I/O操作

我们都了解 NodeJs 是 JavaScript 脱离了浏览器 V8 的执行环境下的另一个 Runtime  ，这也就意味着利用 NodeJS 我们可以进行 I/O 操作（比如从网络读取、访问数据库或文件系统）。

关于 I/O 操作，你可以将它产生的 callback 理解成为 macro 宏任务队列来处理。

### 串联流程

https://juejin.cn/post/7077122129107353636#heading-16

#### Mirco

在主调用栈结束后，会优先处理 prcoess.nextTick 以及之前所有产生的微任务：

<img src="\images\4cd02652b0ea4b5dbb2af53943ba0d94tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp" alt="image.png" style="zoom:50%;" />

正如上图所示，我们可以简单的将 Process.nextTick 以及 micro 统一当作 micro 。

#### timers

之后会正式进入 EventLoop 事件队列，首当其冲的肯定是 timers 定时器 callback 处理阶段：

![image.png](\images\c8585c9ad45743629c870619cef3f99ctplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp)

#### Poll

此后，在清空队列中所有的 timer 后，Loop 进入 poll 阶段进行轮询，此阶段首先会检查是否存在对应 I/O 的callback 。

如果存在 I/O 相关 callback，那么推入对应 JS 调用栈进行执行，同样每次任务执行完毕会伴随清空随之产生的 Process.nextTick 以及 micro 。

当然，如果次阶段即使产生了 timer 也并不会在本次 Loop 中执行，因为此时 EventLoop 已经到达 poll 阶段了。

它会依次去拿出相关的 I/O 回调推入 stack 中进行清空。

需要额外注意的是在 poll 轮询阶段，会发生以下情况：

- *如果 **轮询** 队列 **不是空的*** ，事件循环将循环访问回调队列并同步执行它们，直到队列已用尽，或者达到了与系统相关的硬性限制。
- *如果 **轮询** 队列 **是空的*** ，还有两件事发生：
  - 如果脚本被 `setImmediate()` 调度，则事件循环将结束 **poll(轮询)** 阶段，并继续 **check(检查)** 阶段以执行那些被调度的脚本。
  - 如果脚本 **未被** `setImmediate()`调度，则事件循环将**等待回调**被添加到队列中，然后立即执行(即**阻塞**)。



![image.png](\images\d92ddc0f068148adb741a7164b72267ctplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp)

