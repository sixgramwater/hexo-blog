# 迭代器、生成器、Async

## 迭代协议

### 可迭代协议

### 迭代器协议



## 迭代器 iterator

### 为什么要引入迭代器？

js中有着不同的数据结构，如数组、字符串，ES6中我们又引入了Set、Map等等，同时我们还能组合使用这些数据结构。面对如此众多的数据结构，我们却不能使用一个统一的遍历方式来获取数据，而是需要去考虑采用**不同的遍历方式**。这就是ES6之前存在的问题。

这也是要引入迭代器模式的原因所在。

> 迭代器模式描述了一种方案：将实现了正式的 Iterable 接口，并且可以通过迭代器 Iterator 消费的结构，称为“可迭代对象（iterable）”。

### 什么是可迭代对象（iterable）

一个对象要成为可迭代对象，必须实现`@@iterator`方法，即对象（或者原型链上的某个对象）必须有一个键为 `@@iterator` 的属性，可通过常量 `[Symbol.iterator]`访问该属性。所以：

> **一个对象成为可迭代对象的条件**：实现了`@@iterator`方法

`[Symbol.iterator]`： 一个无参数的函数，其返回值为一个符合**迭代器协议**的对象(即会返回一个迭代器iterator)。

### 什么是迭代器对象

Iterator 必须有 `next()` 方法，它每次返回一个 `{ done: Boolean, value: any }` 对象，这里 `done:true` 表明迭代结束，否则 `value` 就是下一个要迭代的值。

### 手写实现可迭代对象

```js
let iteratorObj = {
    items: [1, 2, 'ljc'],
    // 部署Symbol.iterator属性
    [Symbol.iterator]: function () {
        let self = this;
        let i = 0;
        // 返回一个具有next方法的迭代器对象
        return {
            next: function () {
                // 类型转化为Boolean
                let done = (i >= self.items.length)
                // 数据确认
                let value = !done ? self.items[i++] : undefined
                // 数据返回
                return {
                    done,
                    value
                }
            }
        }
    }
}
for (let item of iteratorObj) {
    console.log(item); // 1 2 ljc
}
```

### Iterator的应用场景

#### 内置可迭代对象

- String
- Array
- TypedArray
- Map
- Set
- 函数的 arguments 对象
- NodeList 等 DOM 集合类型

注意普通的对象则没有默认部署`Iterator`接口。需要我们手动实现。

#### 需要可迭代对象的语法

- for-of
- 解构赋值
- 扩展运算符
- yield*

#### 接收可迭代对象的内置 API

- new Map([iterable])
- new WeakMap([iterable])
- new Set([iterable])
- new WeakSet([iterable])
- Promise.all(iterable)
- Promise.race(iterable)
- Array.from(iterable)

- 自定义可迭代对象



### 提前终止迭代器

迭代器使用**可选**的 return() 方法可以让迭代器提前关闭执行。应用场景包括：

- for-of 循环通过 break\continue\return\throw 等提前退出
- 解构赋值并未消费所有值

将自定义迭代器将入 return() 方法

```javascript
class Counter {
  constructor (limit) {
    this.limit = limit;
  }

  [Symbol.iterator] () {
    let count = 1,
      limit = this.limit;
    return {
      next () {
        if (count <= limit) {
          return { done: false, value: count++ };
        } else {
          return { done: true, value: undefined };
        }
      },
      // 加入 return
      return () {
      	console.log('迭代器关闭前执行操作');
        return { done: true };
      }
    }
  }
}

let counter = new Counter (5);

for (let i of counter) {
  if (i > 2) break;
  console.log(i);
}
// 1
// 2
// 迭代器关闭前执行操作
```

## 生成器 Generator

### 什么是协程

协程是一种比线程更加**轻量**级的存在。你可以把协程看成是跑在线程上的任务，一个线程上可以存在多个协程，但是在线程上同时只能执行一个协程。

协程概念的提出比较早，单核CPU场景中发展出来的概念，通过提供**挂起和恢复**接口，实现在单个CPU上交叉处理多个任务的并发功能。

本质上就是在一个线程的基础上，增加了不同任务栈的切换，通过不同任务栈的挂起和恢复，**线程中进行交替运行的代码片段**，实现并发的功能。（注意是并发不是并行）

### Generator基础入门

所谓 Generator 函数它是**协程**在 ES6 的实现，最大特点就是可以交出函数的执行权（即拥有暂停函数执行的效果）。

#### 1.概念

- **Generator 函数是 ES6 提供的一种异步编程解决方案**，语法行为与传统函数完全不同
- 语法上，首先可以把它理解成，**Generator 函数是一个状态机，封装了多个内部状态**。
- **Generator 函数除了状态机，还是一个遍历器对象生成函数**。
- 可暂停函数(惰性求值), yield可暂停，next方法可启动。每次返回的是yield后的表达式结果

#### 2.特点

- function关键字与函数名之间有一个星号；
- 函数体内部使用`yield`表达式，定义不同的内部状态

```js
 function* generatorExample(){
    console.log("开始执行")
    yield 'hello';  
    yield 'generator'; 
 }
// generatorExample() 
// 这种调用方法Generator 函数并不会执行
let MG = generatorExample() // 返回指针对象
MG.next() //开始执行  {value: "hello", done: false}
```

Generator 函数是分段执行的，调用`next`方法函数内部逻辑开始执行，遇到yield表达式停止，返回`{value: yield后的表达式结果/undefined, done: false/true}`,再次调用`next`方法会从上一次停止时的yield处开始，直到最后。

```js
function* helloWorldGenerator() {
  yield 'hello';
  yield 'world';
  return 'ending';
}
var hw = helloWorldGenerator();
hw.next()// { value: 'hello', done: false }
hw.next()// { value: 'world', done: false }
hw.next()// { value: 'ending', done: true }
hw.next()// { value: undefined, done: true }
```

第一次调用`next`，Generator 函数开始执行，直到遇到第一个`yield`表达式为止。`next`方法返回一个对象，它的`value`属性就是当前`yield`表达式的值hello，`done`属性的值false，表示遍历还没有结束。

第二次调用，Generator 函数从上次`yield`表达式停下的地方，一直执行到下一个`yield`表达式。`next`方法返回的对象的`value`属性就是当前`yield`表达式的值world，`done`属性的值false，表示遍历还没有结束。

第三次调用，Generator 函数从上次`yield`表达式停下的地方，一直执行到return语句（如果没有return语句，就执行到函数结束）。`next`方法返回的对象的value属性，就是紧跟在return语句后面的表达式的值（如果没有return语句，则value属性的值为`undefined`），done属性的值true，表示遍历已经结束。

第四次调用，此时 Generator 函数已经运行完毕，`next`方法返回对象的value属性为`undefined`，done属性为true。以后再调用next方法，返回的都是这个值。

#### 3.next传递参数

`yield`表达式本身没有返回值，或者说总是返回`undefined`。`next`方法可以带一个参数，该参数就会被当作上一个yield表达式的返回值。

```js
function* generatorExample () {
  console.log('开始执行')
  let result = yield 'hello'
  console.log(result)
  yield 'generator'
}
let MG = generatorExample()
MG.next()
MG.next()
// 开始执行
// undefined
// {value: "generator", done: false}
```

没有传值时result默认是`undefined`，接下来我们向第二个next传递一个参数，看下输出结果是啥？

```js
function* generatorExample () {
  console.log('开始执行')
  let result = yield 'hello'
  console.log(result)
  yield 'generator'
}
let MG = generatorExample()
MG.next()
// next(para) => 相应的yield的返回值为para
MG.next(11)
// 开始执行
// 11
// {value: "generator", done: false}
```

### Generator原理实现

https://juejin.cn/post/7069317318332907550#heading-6

![image.png](\images\f3e8dc244d7d44a3bafc49aa7dff689ftplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp)

这是 Babel 在低版本浏览器下为我们实现的 Generator 生成器函数的 [polyfill](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FGlossary%2FPolyfill) 实现。

左侧为 ES6 中的生成器语法，右侧为转译后的兼容低版本浏览器的实现。

首先左侧的 gen 生成器函数被在右侧被转化成为了一个简单的普通函数，具体 gen 函数内容我们先忽略它。

在右侧代码中，对于普通 gen 函数包裹了一层` regeneratorRuntime.mark(gen)` 处理，在源码中这一步其实为了将普通 gen 函数继承 GeneratorFunctionPrototype 从而实现将 gen() 返回的对象变成 Generator 实例对象的处理。

这一步对于我们理解 Generator 显得并不是那么重要，所以我们可以简单的将 `regeneratorRuntime.mark` 改写成为这样的结构:

```js
// 自己定义regeneratorRuntime对象
const regeneratorRuntime = {
    // 存在mark方法，接受传入的fn。原封不懂的返回fn
    mark(fn) {
        return fn
    }
}
```

让我们进入` regeneratorRuntime.wrap` 内部传入的具体函数来看看：

```js
function gen() {
  var a, b, c;
  return regeneratorRuntime.wrap(function gen$(_context) {
    // while(1) 配合函数的return没有任何实际意义
    // 通常在编程中使用 while(1) 来表示while中的内部会被多次执行
    while (1) {
      // 更新_context.prev指向
      switch ((_context.prev = _context.next)) {
        case 0:
          // 更新_context.next指向
          _context.next = 2;
          return 1;

        case 2:
          a = _context.sent;
          console.log(a, 'this is a');
          _context.next = 6;
          return 2;

        case 6:
          b = _context.sent;
          console.log(b, 'this is b');
          _context.next = 10;
          return 3;

        case 10:
          c = _context.sent;
          console.log(c, 'this is c');

        case 12:
        case 'end':
          return _context.stop();
      }
    }
  }, _marked);
}
```

`regeneratorRuntime.wrap` 内部传入的函数，使用了 while(1) 内部逻辑，因为我们在 while 循环中配合了函数的 return 语句，所以这里 while(1) 其实并没有任何含义。

通常，在编程中我们用 while(1) 来表示内部的逻辑会被执行很多次，的确在函数内部的 while 循环每次调用 next 方法其实都会进入这段逻辑执行。

首先我们来看看 传入的 `_context` 参数存在哪些属性：

- `_context.prev` 表示本次生成器函数执行时的指针位置。
- `_context.next` 表示下次生成器函数调用时的指针位置。
- `_context.sent` 表示调用` g.next(params)` 时，传入的 params 参数。
- `_context.stop` 表示当调用 `g.next()` 生成器函数执行完毕调用的方法。

在解释了 `_context` 对象上的属性含义之后，也许你还是不太明白它们各自的含义。我们先来看看简化后的实现代码：

```js
const regeneratorRuntime = {
  // 存在mark方法，接受传入的fn。原封不懂的返回fn
  mark(fn) {
    return fn;
  },
  wrap(fn) {
    const _context = {
      next: 0, // 表示下一次执行生成器函数状态机switch中的下标
      sent: '', // 表示next调用时候传入的值 作为上一次yield返回值
      done: false, // 是否完成
      // 完成函数
      stop() {
        this.done = true;
      },
    };
    return {
      next(param) {
        // 1. 修改上一次yield返回值为context.sent
        _context.sent = param;
        // 2.执行函数 获得本次返回值
        const value = fn(_context);
        // 3. 返回
        return {
          done: _context.done,
          value,
        };
      },
    };
  },
};
```

完整的 `regeneratorRuntime` 对象就像上边实现的那样，它看起来非常简单对吧。

在 `wrap` 函数中，我们接受传入的一个**状态机函数**。每次调用 `wrap()` 方法返回的` next(param)` 方法时，会将 `next(param)` 中传入的参数传递给 `wrap` 函数中的维护的 `_context.sent` 作为模拟上一次 `yield` 返回值而出现。

同时在 `wrap(fn)` 中传入的 `fn` 你可以将它看作**一个小型状态机** ，<u>每次调用 `next()` 方法都会执行状态机 `fn` 函数</u>。

不过，因为状态机中每次执行时 `_context.prev` 的值都是不同的，造成了每次调用` next `函数都执行的是状态机中不同的逻辑。

直到，状态机函数 `fn` 中的 switch 语句匹配完成返回 `_context.stop()` 函数，此时将 `_context.done` 变为 true 并且返回对应对象。

**所谓 Generator 核心代码简化后不过上述短短几十行，它的内部核心思想本质上就是通过 `regeneratorRuntime.wrap` 函数包裹一个状态机函数 fn 。**

**`wrap` 函数内部维护一个 `_context` 对象，从而每次调用返回的生成器对象的 `next` 方法时，被包裹的状态机函数根据 `_context` 的对应属性匹配对应状态来完成不同的逻辑。**




### co模块

```js
function co(gen) {
  return new Promise((resolve, reject) => {
    const g = gen();
    function next(param) {
      const { done, value } = g.next(param);
      if (!done) {
        // 未完成 继续递归
        Promise.resolve(value).then((res) => next(res));
      } else {
        // 完成直接重置 Promise 状态
        resolve(value);
      }
    }
    // 首次执行，无para
    next();
  });
}

co(readFile).then((res) => console.log(res));
```

我们定义了一个 `co` 函数来包裹传入的 generator 生成器函数。

在函数 co 中，我们返回了一个 Promise 来作为包裹函数的返回值，同时首次调用 co 函数时会调用 `gen()` 得到对应的生成器对象。

之后我们定义了一次 `next `方法，在 `next` 函数内部只要迭代器未完成那么此时我们就会在 value 的 `then` 方法中在此**递归调用该 `next` 函数**。

**其实关于异步迭代时，大多数情况下都可以使用类似该函数中的递归方式来处理。**

函数中稍微有三点需要大家额外注意：

- 首先我们可以看到 `next` 函数接受传入的一个 `param` 的参数。

这是因为我们使用 Generator 来处理异步问题时，通过 `const a = yield promise` 将 promise 的 resolve 值交给 a ，所以我们需要在每次 `then` 函数中<u>将 res 传递给下一次的 `next(res)` 作为上次 `yield` 的返回值</u>。

- 其次，细心的同学可以留意到这一句代码`Promise.resolve(value).then((res) => next(res));`。

我们使用 `Promise.resolve` 将 value 进行了一层包裹，这是因为当生成器函数中的 `yield` 方法后紧挨的并不是 Promise 时，此时我们需要统一当作 Promise 来处理，因为我们需要统一调用 `.then` 方法。

- 最后，首次调用 `next()` 方法时，我们并没有传入 param 参数。

相信这个并不难理解，当我们不传入 param 时相当于直接调用 `g.next()` ，上边我们提到过当调用生成器对象的 `next` 方法传入参数时，该参数会当作上一次 yield 语句的返回值来处理。

因为首次调用 `g.next()` 时，生成器函数内部之前并不存在 `yield` ，所以传入参数是没有任何意义的。

它看来并不是很难对吧，但是通过这一小段代码让我们的 Generator 拥有了可以让异步代码同步调用的书写方式来使用。

其实这一小段代码也是所谓 [co](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Ftj%2Fco) 库的核心原理，当然所谓 co 远远不止这些，但是这段代码足够我们了解所谓在 Async/Await 未出现之前我们是如何使用所谓的 Generator 来作为终极异步解决方案了。

## Async, Await