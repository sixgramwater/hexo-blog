## 函数对象 NFE

### name

### length

### arguments

arguments是一个**类数组对象**，在函数调用时创建，它存储的是实际传递给函数的参数，并不局限于函数定义时的参数列表。
`类数组对象特点: 具有index和length属性，但不具有数组的操作方法，比如push，shift等。`



### callee

### caller



## 类数组对象(array-like)

## 深入理解箭头函数



## 对象-原始值转换(Object -> Primitive)

https://zh.javascript.info/object-toprimitive

## 对象属性配置

### 属性标志和属性描述符

https://zh.javascript.info/property-descriptors

<img src="\images\c20c73cca5cd5dbbbbce823ad7cb9c05954a87ecb2aca9f927d150de8eba3c4e-16565040796162.png" alt="img" style="zoom: 150%;" />

#### 属性标志 

对象属性（properties），除 **`value`** 外，还有三个特殊的特性（attributes），也就是所谓的“标志”：

- **`writable`** — 如果为 `true`，则值可以被修改，否则它是只可读的。
- **`enumerable`** — 如果为 `true`，则会被在循环中列出，否则不会被列出。
- **`configurable`** — 如果为 `true`，则此属性可以被删除，这些特性也可以被修改，否则不可以。

我们到现在还没看到它们，是因为它们通常不会出现。当我们用“常用的方式”创建一个属性时，它们都为 `true`。但我们也可以随时更改它们。

[Object.getOwnPropertyDescriptor](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertyDescriptor) 方法允许查询有关属性的 **完整** 信息。

语法是：

```javascript
let descriptor = Object.getOwnPropertyDescriptor(obj, propertyName);
```

- `obj`

  需要从中获取信息的对象。

- `propertyName`

  属性的名称。

返回值是一个所谓的“属性描述符”对象：它包含值和所有的标志。

例如：

```javascript
let user = {
  name: "John"
};

let descriptor = Object.getOwnPropertyDescriptor(user, 'name');

alert( JSON.stringify(descriptor, null, 2 ) );
/* 属性描述符：
{
  "value": "John",
  "writable": true,
  "enumerable": true,
  "configurable": true
}
*/
```

为了修改标志，我们可以使用 [Object.defineProperty](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)。

语法是：

```javascript
Object.defineProperty(obj, propertyName, descriptor)
```

- `obj`，`propertyName`

  要应用描述符的对象及其属性。

- `descriptor`

  要应用的属性描述符对象。

如果该属性存在，`defineProperty` 会更新其标志。否则，它会使用给定的值和标志创建属性；在这种情况下，如果没有提供标志，则会假定它是 `false`。

例如，这里创建了一个属性 `name`，该属性的所有标志都为 `false`：

```javascript
let user = {};

Object.defineProperty(user, "name", {
  value: "John"
});

let descriptor = Object.getOwnPropertyDescriptor(user, 'name');

alert( JSON.stringify(descriptor, null, 2 ) );
/*
{
  "value": "John",
  "writable": false,
  "enumerable": false,
  "configurable": false
}
 */
```

将它与上面的“以常用方式创建的” `user.name` 进行比较：现在所有标志都为 `false`。如果这不是我们想要的，那么我们最好在 `descriptor` 中将它们设置为 `true`。

现在让我们通过示例来看看标志的影响。

### 属性的 getter 和 setter

#### 数据属性与`访问器属性`

有两种类型的对象属性。

第一种是 **数据属性**。我们已经知道如何使用它们了。到目前为止，我们使用过的所有属性都是数据属性(有value)。

第二种类型的属性是新东西。它是 **访问器属性（accessor property）**。它们本质上是用于获取和设置值的函数，但从外部代码来看就像常规属性。（没有value）

访问器属性由 “getter” 和 “setter” 方法表示。在对象字面量中，它们用 `get` 和 `set` 表示：

```javascript
let obj = {
  get propName() {
    // 当读取 obj.propName 时，getter 起作用
  },

  set propName(value) {
    // 当执行 obj.propName = value 操作时，setter 起作用
  }
};
```

当读取 `obj.propName` 时，getter 起作用，当 `obj.propName` 被赋值时，setter 起作用。

例如，我们有一个具有 `name` 和 `surname` 属性的对象 `user`：

```javascript
let user = {
  name: "John",
  surname: "Smith"
};
```

现在我们想添加一个 `fullName` 属性，该属性值应该为 `"John Smith"`。当然，我们不想复制粘贴已有的信息，因此我们可以使用访问器来实现：

```javascript
let user = {
  name: "John",
  surname: "Smith",

  get fullName() {
    return `${this.name} ${this.surname}`;
  }
};

alert(user.fullName); // John Smith
```

从外表看，访问器属性看起来就像一个普通属性。这就是访问器属性的设计思想。我们不以函数的方式 **调用** `user.fullName`，我们正常 **读取** 它：getter 在幕后运行。

截至目前，`fullName` 只有一个 getter。如果我们尝试赋值操作 `user.fullName=`，将会出现错误：

```javascript
let user = {
  get fullName() {
    return `...`;
  }
};

user.fullName = "Test"; // Error（属性只有一个 getter）
```

让我们通过为 `user.fullName` 添加一个 setter 来修复它：

```javascript
let user = {
  name: "John",
  surname: "Smith",

  get fullName() {
    return `${this.name} ${this.surname}`;
  },

  set fullName(value) {
    [this.name, this.surname] = value.split(" ");
  }
};

// set fullName 将以给定值执行
user.fullName = "Alice Cooper";

alert(user.name); // Alice
alert(user.surname); // Cooper
```

现在，我们就有一个“虚拟”属性。它是可读且可写的。

#### 访问器描述符

访问器属性的描述符与数据属性的不同。

对于访问器属性，没有 `value` 和 `writable`，但是有 `get` 和 `set` 函数。

所以访问器描述符可能有：

- **`get`** —— 一个没有参数的函数，在读取属性时工作，
- **`set`** —— 带有一个参数的函数，当属性被设置时调用，
- **`enumerable`** —— 与数据属性的相同，
- **`configurable`** —— 与数据属性的相同。

例如，要使用 `defineProperty` 创建一个 `fullName` 访问器，我们可以使用 `get` 和 `set` 来传递描述符：

```javascript
let user = {
  name: "John",
  surname: "Smith"
};

Object.defineProperty(user, 'fullName', {
  get() {
    return `${this.name} ${this.surname}`;
  },

  set(value) {
    [this.name, this.surname] = value.split(" ");
  }
});

alert(user.fullName); // John Smith

for(let key in user) alert(key); // name, surname
```

请注意，一个属性要么是访问器（具有 `get/set` 方法），要么是数据属性（具有 `value`），但不能两者都是。

如果我们试图在同一个描述符中同时提供 `get` 和 `value`，则会出现错误：

```javascript
// Error: Invalid property descriptor.
Object.defineProperty({}, 'prop', {
  get() {
    return 1
  },

  value: 2
});
```

## Promise 为什么这么设计？

promise为什么要引入微任务，

除了解决回调地狱问题，它还解决了什么？

## V8- JS执行流程

https://segmentfault.com/a/1190000039380905



https://juejin.cn/post/6844904137792962567



https://v8.dev/blog/preparser

![img](\images\171fdeedf49c5874tplv-t2oaga2asx-zoom-in-crop-mark3024000.awebp)

我们来一起看一下V8现有架构是如何执行js代码的：

- 第一步，将js源代码转化成AST（抽象语法树）
- 第二步，通过Ignition将AST编译成字节码，开始逐句对字节码进行解释成二进制代码并执行。
- 第三步，在解释执行的过程中，标记重复执行的热点代码，将标记的代码通过Turbofan引擎进行编译生成效率更高二进制代码，再次运行到这个函数时便只执行高效代码而不再解释执行字节码。

V8引入了字节码的架构模式后明显的解决了如下问题：

- 启动时间较长：启动时只需要编译出字节码，然后逐句执行字节码，编译出字节码的速度可远远快于编译出二进制代码的速度。
- 内存占用较大：字节码的空间占用也是远远低于二进制代码的空间占用。
- 代码复杂度太高：大大降低了V8适应不同CPU所需要的代码复杂程度。



### Parser

![image-20220704102321736](\images\image-20220704102321736.png)

#### Two Parsers

**Parser**: full, eager

- used for parsing functions we want to compile
- Build AST
- Build Scopes
- Finds all syntax errors

**PreParser**: fast, Lazy

- Used for skipping over functions which we dont want to compile
- Doesn't build AST; build Scopes but doesnt put variable refernce or declarations in them.
- twice as fast as Parser

#### Parser

![image-20220704103856260](\images\image-20220704103856260.png)



#### PreParser

Lazy parsing



### Ignition

### TurboFan

## 模块化

https://segmentfault.com/a/1190000023711059

### CJS

在 Node.js 模块系统中，每个文件都被视为一个单独的模块，在一个Node.js 的模块中，本地的变量是私有的，而这个私有的实现，是通过把 Node.js 的模块包装在一个函数中，也就是 `The module wrapper`，我们来看看，在 [官方示例中](https://link.juejin.cn?target=https%3A%2F%2Fnodejs.org%2Fdocs%2Flatest%2Fapi%2Fmodules.html%23modules_the_module_wrapper) 它长什么样：

```js
(function(exports, require, module, __filename, __dirname) {
// Module code actually lives in here
// 实际上，模块内的代码被放在这里
});
```

是的，在模块内的代码被真正执行以前，实际上，这些代码都被包含在了一个这样的函数中。

如果你真正阅读了上一节中关于 IIFE 的内容，你会发现，其实核心思想是一样的，Node.js 对于模块私有化的实现也还是通过了一个函数。但是这有哪些不同呢？

虽然这里有 5 个参数，但是我们把它们先放在一边，然后尝试站在一个模块的角度来思考这样一个问题：作为一个模块，你希望自己具备什么样的能力呢?

1. **暴露部分自己的方法或者变量的能力** ：这是我存在的意义，因为，对于那些想使用我的人而言这是必须的。[ `exports:导出对象` , `module:模块的引用` ]
2. **引入其他模块的能力**：有的时候我也需要通过别人的帮助来实现一些功能，只把我的注意力放在我想做的事情（核心逻辑）上。[ `require:引用方法` ]
3. **告诉别人我的物理位置**：方便别人找到我，并且对我进行更新或者修改。[ `__filename:绝对文件名`, `__dirname:目录路径` ]

#### CJS中`require`的实现

为什么我们要了解 `require` 方法的实现呢？因为理解这一过程，我们可以更好地理解下面的几个问题：

1. 当我们引入一个模块的时候，我们究竟做了怎样一件事情？
2. `exports` 和 `module.exports` 有什么联系和区别？
3. 这样的方式有什么弊端？

在文档中，有简易版的 `require` 的实现：

```js
function require(/* ... */) {
  const module = { exports: {} };
  ((module, exports) => {
    // Module code here. In this example, define a function.
    // 模块代码在这里，在这个例子中，我们定义了一个函数
    function someFunc() {}
    exports = someFunc;
    // At this point, exports is no longer a shortcut to module.exports, and
    // this module will still export an empty default object.
    // 当代码运行到这里时，exports 不再是 module.exports 的引用，并且当前的
    // module 仍旧会导出一个空对象(就像上面声明的默认对象那样)
    module.exports = someFunc;
    // At this point, the module will now export someFunc, instead of the
    // default object.
    // 当代码运行到这时，当前 module 会导出 someFunc 而不是默认的对象
  })(module, module.exports);
  return module.exports;
}
```

回到刚刚提出的问题：

**1. `require` 做了怎样一件事情?**

require 相当于把被引用的 module 拷贝了一份到当前 module 中

**2. `exports` 和 `module.exports` 的联系和区别？**

代码中的注释以及 require 函数第一行默认值的声明，很清楚的阐述了，`exports` 和 `module.exports` 的区别和联系:

`exports` 是 `module.exports` 的引用。作为一个引用，如果我们修改它的值，实际上修改的是它对应的引用对象的值。

就如:

```js
exports.a = 1
// 等同于
module.exports = {
    a: 1
}
```

但是如果我们修改了 exports 引用的地址，对于它原来所引用的内容来说，没有任何影响，反而我们断开了这个引用于原来的地址之间的联系：

```js
exports = {
    a: 1
}

// 相当于

let other = {a: 1} //为了更加直观，我们这样声明了一个变量
exports = other
```

`exports` 从指向 `module.exports` 变为了 `other`。

**3. 弊端**

`CommonJS` 这一标准的初衷是为了让 `JavaScript` 在多个环境下都实现模块化，但是 Node.js 中的实现依赖了 Node.js 的环境变量：`module`，`exports`，`require`，`global`，浏览器没法用啊，所以后来出现了 `Browserify` 这样的实现，但是这并不是本文要讨论的内容，有兴趣的同学可以读读阮一峰老师的 [这篇文章](https://link.juejin.cn?target=http%3A%2F%2Fwww.ruanyifeng.com%2Fblog%2F2015%2F05%2Fcommonjs-in-browser.html)。

说完了服务端的模块化，接下来我们聊聊，在浏览器这一端的模块化，又经历了些什么呢？

### CJS的Require原理

#### require的加载逻辑

<img src="http://cdn.ayqy.net/data/home/qxu1001840309/htdocs/cms/wordpress/wp-content/uploads/2020/05/js-module-496x1024.jpg" alt="js module" style="zoom:80%;" />

注意一个细节，是在加载&执行模块文件前会**先缓存**`module`实例，而不是之后才缓存，这是*Node.js 能够从容应对**循环依赖**的根本原因*：

> 举例：A require B, B require A。此时A未执行完，调用了require,但是由于提前缓存了A，导致在B中执行require A时，已经有A的（未完成的）缓存了。这就解决了循环依赖的问题。

如果模块加载过程中出现了循环引用，导致尚未加载完成的模块被引用到，按照图示的模块加载流程也会命中缓存（而不至于进入死递归），即便此时的`module.exports`可能不完整（模块代码没执行完，有些东西还没挂上去）

### RequreJS与AMD(Asynchronous Module Definition)

#### 背景

浏览器端借助Browserify的方式用类似Node.js的方式来管理我们的模块，会有什么问题？

因为我们已经了解了 `require()` 的实现，所以你会发现这其实是一个复制的过程，将被 require 的内容，赋值到一个 module 对象的属性上，然后返回这个对象的 exports 属性。

这样做会有什么问题呢？在我们还没有完成复制的时候，无法使用被引用的模块中的方法和属性。在服务端可能这不是一个问题(因为服务器的文件都是存放在本地，并且是有缓存的)，但在浏览器环境下，这会导致阻塞，使得我们后面的步骤无法进行下去，还可能会执行一个未定义的方法而导致出错。

相对于服务端的模块化，浏览器环境下，模块化的标准必须满足一个新的需求：***异步**的模块管理*

> 异步模块定义规范（AMD）制定了定义模块的规则，这样模块和模块的依赖可以被异步加载。这和浏览器的异步加载模块的环境刚好适应（浏览器同步加载模块会导致性能、可用性、调试和跨域访问等问题）。

本规范只定义了一个函数`define`，它是全局变量。

```typescript
/**
 * @param {string} id 模块名称
 * @param {string[]} dependencies 模块所依赖模块的数组
 * @param {function} factory 模块初始化要执行的函数或对象
 * @return {any} 模块导出的接口
 */
function define(id?, dependencies?, factory): any
```

AMD 是一种异步模块规范，RequireJS 是 AMD 规范的实现。

#### RequireJs

官方文档中的使用的例子：

```js
requirejs.config({
    // 默认加载 js/lib 路径下的module ID
    baseUrl: 'js/lib',
    // 除去 module ID 以 "app" 开头的 module 会从 js/app 路径下加载。
    // 关于 paths 的配置是与 baseURL 关联的，并且因为 paths 可能会是一个目录，
    // 所以不要使用 .js 扩展名 
    paths: {
        app: '../app'
    }
});

// 开始主逻辑
requirejs(['jquery', 'canvas', 'app/sub'],
function   ($,        canvas,   sub) {
    //jQuery, canvas 和 app/sub 模块已经被加载并且可以在这里使用了。
});
```


官方文档中的定义的例子：

```js
// 简单的对象定义
define({
    color: "black",
    size: "unisize"
});

// 当你需要一些逻辑来做准备工作时可以这样定义：
define(function () {
    //这里可以做一些准备工作
    return {
        color: "black",
        size: "unisize"
    }
});

// 依赖于某些模块来定义属于你自己的模块
define(["./cart", "./inventory"], function(cart, inventory) {
        //通过返回一个对象来定义你自己的模块
        return {
            color: "blue",
            size: "large",
            addToCart: function() {
                inventory.decrement(this);
                cart.add(this);
            }
        }
    }
);
```



#### 优势

- 以**函数**的形式返回模块的值，尤其是构造函数，可以更好的实现API 设计，Node 中通过 `module.exports` 来支持这个，但使用 `return function (){}` 会更清晰。 这意味着，我们不必通过处理 “module” 来实现 “module.exports”，它是一个更清晰的代码表达式。
- 动态代码加载
- https://juejin.cn/post/6844903829448687624#heading-12

#### 新的问题 - 依赖前置

通过上面的语法说明，我们会发现一个很明显的问题，在使用 RequireJS 声明一个模块时，必须指定所有的依赖项 ，这些依赖项会被当做形参传到 factory 中，对于依赖的模块会提前执行（在 RequireJS 2.0 也可以选择延迟执行），这被称为：**依赖前置**。

这会带来什么问题呢？

加大了开发过程中的难度，无论是阅读之前的代码还是编写新的内容，也会出现这样的情况：引入的另一个模块中的内容是条件性执行的。

### SeaJS与CMD

针对 AMD 规范中可以优化的部分，[CMD 规范](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fcmdjs%2Fspecification%2Fblob%2Fmaster%2Fdraft%2Fmodule.md) 出现了，而 [SeaJS](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fseajs%2Fseajs) 则作为它的具体实现之一，与 AMD 十分相似：

```js
// AMD 的一个例子，当然这是一种极端的情况
define(["header", "main", "footer"], function(header, main, footer) { 
    if (xxx) {
      header.setHeader('new-title')
    }
    if (xxx) {
      main.setMain('new-content')
    }
    if (xxx) {
      footer.setFooter('new-footer')
    }
});

 // 与之对应的 CMD 的写法
define(function(require, exports, module) {
    if (xxx) {
      var header = require('./header')
      header.setHeader('new-title')
    }
    if (xxx) {
      var main = require('./main')
      main.setMain('new-content')
    }
    if (xxx) {
      var footer = require('./footer')
      footer.setFooter('new-footer')
    }
});
```

我们可以很清楚的看到，CMD 规范中，只有当我们用到了某个外部模块的时候，它才会去引入，这回答了我们上一小节中遗留的问题，这也是它与 AMD 规范最大的不同点：*CMD推崇**依赖就近 + 延迟执行***



#### 仍然存在的问题

我们能够看到，按照 CMD 规范的依赖就近的规则定义一个模块，会导致模块的加载逻辑偏重，有时你并不知道当前模块具体依赖了哪些模块或者说这样的依赖关系并不直观。

而且对于 AMD 和 CMD 来说，都只是适用于浏览器端的规范，而 CJS 仅仅适用于服务端，都有各自的局限性。



### ESM



### CJS与ESM的对比

CommonJS 和 AMD 是运行时加载，在运行时确定模块的依赖关系。
ES6 module 是在编译时（`import()`是运行时加载）处理模块依赖关系

#### 引用vs拷贝

CJS模块`require`导入的是值的**拷贝**，而ESM导入的是值的**引用**。

```javascript
// a.cjs
let age = 18;

exports.setAge = function setAge(val) {
  age = val;
};
exports.age = age;

// index.cjs
const { age, setAge } = require('./a.cjs');

console.log(age); // 18
setAge(19);
console.log(age); // 18

// a.mjs
export let age = 18;
export function setAge(val) {
  age = val;
}

// index.mjs
import { age, setAge } from './a.mjs';

console.log(age); // 18
setAge(19);
console.log(age); // 19
```

可以看到，`index.cjs`从`a.cjs`引入了`age`，并通过`setAge`修改了`a.cjs`里的`age`，但是最后打印的`age`没有变，而ESM则相反。

> 注意到对于原始值来说也能够做到引用传递。
>
> ESM的这一特性在webpack中是如何实现的呢？
>
> -- 通过`Object.defineProperties`来创建一个getter

另外，由于ESM是输出值的引用，所以**不允许**在外部直接修改值（对象修改或新增属性除外），否则报错；

#### 动态vs静态

我们都知道javascript是一门JIT语言，v8引擎拿到js代码后会边编译边执行，在编译的时候v8就给`import`导入的模块建立静态的引用，并且不能在运行时不能更改。所以`import`都放在文件开头，不能放在条件语句里。 

而`require`导入模块是在运行时才对值进行拷贝，所以`require`的路径可以使用变量，并且`require`可以放在代码的任何位置。 

基于这个差异，ESM比CJS好做tree-shaking。

#### 异步vs同步

由于CJS是用于服务器端的模块体系，需要加载的模块都在本地，所以采用同步加载也不会出问题，但是ESM用于浏览器端时，可能涉及到一些异步请求，所以需要采用异步加载。


ESM是顶层await的设计，而CJS中的require是同步加载，所以require无法导入ESM模块，但是可以通过`import()`导入。

#### web项目中ESM的处理

我们平时用react、vue开发业务的时候都是遵循ESM规范，但最终交给浏览器执行的并不是ESM的代码，因为需要兼容旧版本的浏览器嘛。处理过程大致如下：

1. ESM规范编写代码，使用`import`、`export`;
2. babel等编译器将ESM代码转成CJS代码；
3. 但是浏览器不支持CJS规范啊，所以webpack按照CJS规范实现了类似`require`和`module.exports`的模块加载机制。

> 这里顺便说一下最近比较热门的话题：esbuild 0.14.4版本在CJS和ESM的转换上引入了breaking change，掀起社区热烈的讨论，esbuild也在[changelog](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fevanw%2Fesbuild%2Fblob%2Fmaster%2FCHANGELOG.md%230144)里详细记录了事情的来由。大概情况就是babel为了将ESM准确降级成CJS，把`export default 0`处理成`module.exports.default = 0`，然后通过`__esModule`是否为true决定`import foo from 'bar'`时foo是`module.exports.default`还是`module.exports`来保证`import foo from 'bar'`和`const foo = require('bar')`等价。但是nodejs ESM的实现是将`export default`和`module.exports`对等起来。这种不一致导致esbuild对nodejs和browser这两个环境下使用的三方库的处理出现错误。




### CJS、ESM的循环依赖问题

首先来看**CJS**

注意一个细节，是在加载&执行模块文件前会**先缓存**`module`实例，而不是之后才缓存，这是*Node.js 能够从容应对**循环依赖**的根本原因*：

> 举例：A require B, B require A。此时A未执行完，调用了require,但是由于提前缓存了A，导致在B中执行require A时，已经有A的（未完成的）缓存了。这就解决了循环依赖的问题。

如果模块加载过程中出现了循环引用，导致尚未加载完成的模块被引用到，按照图示的模块加载流程也会命中缓存（而不至于进入死递归），即便此时的`module.exports`可能不完整（模块代码没执行完，有些东西还没挂上去）



再来看**ESM**

ES6模块的运行机制与CommonJS不一样，它遇到模块加载命令`import`时，不会去执行模块，而是只生成一个引用。等到真的需要用到时，再到模块里面去取值。

因此，ES6模块是***动态引用**，不存在缓存值的问题*，而且模块里面的变量，绑定其所在的模块。

这导致ES6处理"循环加载"与CommonJS有本质的不同。**ES6根本不会关心是否发生了"循环加载"，只是生成一个指向被加载模块的引用，需要开发者自己保证，真正取值的时候能够取到值。**

请看下面的例子（摘自 Dr. Axel Rauschmayer 的[《Exploring ES6》](http://exploringjs.com/es6/ch_modules.html)）。

```javascript
// a.js
import {bar} from './b.js';
export function foo() {
  bar();  
  console.log('执行完毕');
}
foo();

// b.js
import {foo} from './a.js';
export function bar() {  
  if (Math.random() > 0.5) {
    foo();
  }
}
```

按照CommonJS规范，上面的代码是没法执行的。`a`先加载`b`，然后`b`又加载`a`，这时`a`还没有任何执行结果，所以输出结果为`null`，即对于`b.js`来说，变量`foo`的值等于`null`，后面的`foo()`就会报错。

## Rollup

## 浏览器多页面通信

https://juejin.cn/post/6844903811232825357



### 同源页面间的跨页面通信

共有三类不同思路：

1. 广播模式：一个页面将消息通知给一个“中央站”，再由“中央站”通知给各个页面。
2. 共享存储+轮询模式
3. 

#### 1.broadcastCannel

> [BroadCast Channel](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fen-US%2Fdocs%2FWeb%2FAPI%2FBroadcastChannel) 可以帮我们创建一个用于广播的通信频道。当所有页面都监听同一频道的消息时，其中某一个页面通过它发送的消息就会被其他所有页面收到。它的API和用法都非常简单。

下面的方式就可以创建一个标识为`AlienZHOU`的频道：

```js
const bc = new BroadcastChannel('AlienZHOU');
```

各个页面可以通过`onmessage`来监听被广播的消息：

```js
bc.onmessage = function (e) {
    const data = e.data;
    const text = '[receive] ' + data.msg + ' —— tab ' + data.from;
    console.log('[BroadcastChannel] receive message:', text);
};
```

要发送消息时只需要调用实例上的`postMessage`方法即可：

```js
bc.postMessage(mydata);
```



#### 2.localStorage

> LocalStorage 作为前端最常用的本地存储，大家应该已经非常熟悉了；但[`StorageEvent`](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fen-US%2Fdocs%2FWeb%2FAPI%2FStorageEvent)这个与它相关的事件有些同学可能会比较陌生。
>
> 当 LocalStorage 变化时，会触发`storage`事件。利用这个特性，我们可以在发送消息时，把消息写入到某个 LocalStorage 中；然后在各个页面内，通过监听`storage`事件即可收到通知。(需要注意storage事件只有在值改变时才会触发，设置和原来相同的值是不会触发的)

```js
window.addEventListener('storage', function (e) {
    if (e.key === 'ctc-msg') {
        const data = JSON.parse(e.newValue);
        const text = '[receive] ' + data.msg + ' —— tab ' + data.from;
        console.log('[Storage I] receive message:', text);
    }
});
```

在各个页面添加如上的代码，即可监听到 LocalStorage 的变化。当某个页面需要发送消息时，只需要使用我们熟悉的`setItem`方法即可：

```js
mydata.st = +(new Date);
window.localStorage.setItem('ctc-msg', JSON.stringify(mydata));
```

注意，这里有一个细节：我们在mydata上添加了一个取当前毫秒时间戳的`.st`属性。这是因为，`storage`事件只有**在值真正改变时才会触发**。举个例子：

```js
window.localStorage.setItem('test', '123');
window.localStorage.setItem('test', '123');
```

由于第二次的值`'123'`与第一次的值相同，所以以上的代码只会在第一次`setItem`时触发`storage`事件。因此我们通过设置`st`来保证每次调用时一定会触发`storage`事件。



#### 3.serviceWorker

> [Service Worker](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FWeb%2FAPI%2FService_Worker_API) 是一个可以长期运行在后台的 Worker，能够实现与页面的双向通信。多页面共享间的 Service Worker 可以共享，将 Service Worker 作为消息的处理中心（中央站）即可实现广播效果。

serviceWorker的注册略

Service Worker 本身并不自动具备“广播通信”的功能，需要我们添加些代码，将其改造成消息中转站：

```js
/* ../util.sw.js Service Worker 逻辑 */
self.addEventListener('message', function (e) {
    console.log('service worker receive message', e.data);
    e.waitUntil(
        // sw可以通过clients拿到所有打开的标签页
        self.clients.matchAll().then(function (clients) {
            if (!clients || clients.length === 0) {
                return;
            }
            // postMessage转发给每个client
            clients.forEach(function (client) {
                client.postMessage(e.data);
            });
        })
    );
});
```

我们在 Service Worker 中监听了`message`事件，获取页面（从 Service Worker 的角度叫 client）发送的信息。然后通过`self.clients.matchAll()`获取当前注册了该 Service Worker 的所有页面，通过调用每个client（即页面）的`postMessage`方法，向页面发送消息。这样就把从一处（某个Tab页面）收到的消息通知给了其他页面。

处理完 Service Worker，我们需要在页面监听 Service Worker 发送来的消息：(页面收消息)

```js
/* 页面逻辑 */
navigator.serviceWorker.addEventListener('message', function (e) {
    const data = e.data;
    const text = '[receive] ' + data.msg + ' —— tab ' + data.from;
    console.log('[Service Worker] receive message:', text);
});
```

最后，当需要同步消息时，可以调用 Service Worker 的`postMessage`方法：(页面发消息)

```js
/* 页面逻辑 */
navigator.serviceWorker.controller.postMessage(mydata);
```





上面我们看到了三种实现跨页面通信的方式，不论是建立广播频道的 Broadcast Channel，还是使用 Service Worker 的消息中转站，抑或是些 tricky 的`storage`事件，其都是“**广播模式**”：一个页面将消息通知给一个“中央站”，再由“中央站”通知给各个页面。

> 在上面的例子中，这个“中央站”可以是一个 BroadCast Channel 实例、一个 Service Worker 或是 LocalStorage。

下面我们会看到另外两种跨页面通信方式，我把它称为“共享存储+轮询模式”。



#### 4.sharedWorker

> [Shared Worker](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fen-US%2Fdocs%2FWeb%2FAPI%2FSharedWorker) 是 Worker 家族的另一个成员。普通的 Worker 之间是独立运行、数据互不相通；而多个 Tab 注册的 Shared Worker 则可以实现数据共享。

Shared Worker 在实现跨页面通信时的问题在于，它**无法主动通知所有页面**，因此，我们会使用**轮询**的方式，来**拉取**最新的数据。思路如下：

让 Shared Worker 支持两种消息。一种是 post，Shared Worker 收到后会将该数据保存下来；另一种是 get，Shared Worker 收到该消息后会将保存的数据通过`postMessage`传给注册它的页面。也就是让页面通过 get 来主动获取（同步）最新消息。具体实现如下：

首先，我们会在页面中启动一个 Shared Worker，启动方式非常简单：

```js
// 构造函数的第二个参数是 Shared Worker 名称，也可以留空
const sharedWorker = new SharedWorker('../util.shared.js', 'ctc');
```

然后，在该 Shared Worker 中支持 get 与 post 形式的消息：

```js
/* ../util.shared.js: Shared Worker 代码 */
let data = null;
self.addEventListener('connect', function (e) {
    const port = e.ports[0];
    port.addEventListener('message', function (event) {
        // get 指令则返回存储的消息数据
        if (event.data.get) {
            data && port.postMessage(data);
        }
        // 非 get 指令则存储该消息数据
        else {
            data = event.data;
        }
    });
    port.start();
});
```

之后，页面定时发送 get 指令的消息给 Shared Worker，轮询最新的消息数据，并在页面监听返回信息：

```js
// 定时轮询，发送 get 指令的消息
setInterval(function () {
    sharedWorker.port.postMessage({get: true});
}, 1000);

// 监听 get 消息的返回数据
sharedWorker.port.addEventListener('message', (e) => {
    const data = e.data;
    const text = '[receive] ' + data.msg + ' —— tab ' + data.from;
    console.log('[Shared Worker] receive message:', text);
}, false);
sharedWorker.port.start();
```

最后，当要跨页面通信时，只需给 Shared Worker `postMessage`即可：

```js
sharedWorker.port.postMessage(mydata);
```

> 注意，如果使用`addEventListener`来添加 Shared Worker 的消息监听，需要显式调用`MessagePort.start`方法，即上文中的`sharedWorker.port.start()`；如果使用`onmessage`绑定监听则不需要。



#### 5.IndexedDB

除了可以利用 Shared Worker 来共享存储数据，还可以使用其他一些“全局性”（支持跨页面）的存储方案。例如 [IndexedDB](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fen-US%2Fdocs%2FWeb%2FAPI%2FIndexedDB_API) 或 cookie。

鉴于大家对 cookie 已经很熟悉，加之作为“互联网最早期的存储方案之一”，cookie 已经在实际应用中承受了远多于其设计之初的责任，我们下面会使用 IndexedDB 来实现

其思路很简单：与 Shared Worker 方案类似，消息发送方将消息存至 IndexedDB 中；接收方（例如所有页面）则通过轮询去获取最新的信息。在这之前，我们先简单封装几个 IndexedDB 的工具方法。

- 打开数据库连接

```js
function openStore() {
    const storeName = 'ctc_aleinzhou';
    return new Promise(function (resolve, reject) {
        if (!('indexedDB' in window)) {
            return reject('don\'t support indexedDB');
        }
        const request = indexedDB.open('CTC_DB', 1);
        request.onerror = reject;
        request.onsuccess =  e => resolve(e.target.result);
        request.onupgradeneeded = function (e) {
            const db = e.srcElement.result;
            if (e.oldVersion === 0 && !db.objectStoreNames.contains(storeName)) {
                const store = db.createObjectStore(storeName, {keyPath: 'tag'});
                store.createIndex(storeName + 'Index', 'tag', {unique: false});
            }
        }
    });
}

```

- 存储数据

```js
function saveData(db, data) {
    return new Promise(function (resolve, reject) {
        const STORE_NAME = 'ctc_aleinzhou';
        const tx = db.transaction(STORE_NAME, 'readwrite');
        const store = tx.objectStore(STORE_NAME);
        const request = store.put({tag: 'ctc_data', data});
        request.onsuccess = () => resolve(db);
        request.onerror = reject;
    });
}
```

- 查询/读取数据

```js
function query(db) {
    const STORE_NAME = 'ctc_aleinzhou';
    return new Promise(function (resolve, reject) {
        try {
            const tx = db.transaction(STORE_NAME, 'readonly');
            const store = tx.objectStore(STORE_NAME);
            const dbRequest = store.get('ctc_data');
            dbRequest.onsuccess = e => resolve(e.target.result);
            dbRequest.onerror = reject;
        }
        catch (err) {
            reject(err);
        }
    });
}
```



剩下的工作就非常简单了。首先打开数据连接，并初始化数据：

```js
openStore().then(db => saveData(db, null))
```

对于消息读取，可以在连接与初始化后轮询：

```js
openStore().then(db => saveData(db, null)).then(function (db) {
    setInterval(function () {
        query(db).then(function (res) {
            if (!res || !res.data) {
                return;
            }
            const data = res.data;
            const text = '[receive] ' + data.msg + ' —— tab ' + data.from;
            console.log('[Storage I] receive message:', text);
        });
    }, 1000);
});
```

最后，要发送消息时，只需向 IndexedDB 存储数据即可：

```js
openStore().then(db => saveData(db, null)).then(function (db) {
    // …… 省略上面的轮询代码
    // 触发 saveData 的方法可以放在用户操作的事件监听内
    saveData(db, mydata);
});
```



> 在“广播模式”外，我们又了解了“共享存储+长轮询”这种模式。也许你会认为长轮询没有监听模式优雅，但实际上，有些时候使用“共享存储”的形式时，不一定要搭配长轮询。(取决于你希望同步消息的时机和频率)
>
> 例如，在多 Tab 场景下，我们可能会离开 Tab A 到另一个 Tab B 中操作；过了一会我们从 Tab B 切换回 Tab A 时，希望将之前在 Tab B 中的操作的信息同步回来。这时候，其实只用在 Tab A 中监听`visibilitychange`这样的事件，来做一次信息同步即可。

下面，我会再介绍一种通信方式，我把它称为“口口相传”模式。



#### 6. window.open + window.opener

当我们使用`window.open`打开页面时，方法会返回一个被打开页面`window`的引用。而在未显示指定`noopener`时，被打开的页面可以通过`window.opener`获取到打开它的页面的引用 —— 通过这种方式我们就将这些页面建立起了联系（一种**树形结构**）。

首先，我们把`window.open`打开的页面的`window`对象收集起来：

```js
let childWins = [];
document.getElementById('btn').addEventListener('click', function () {
    const win = window.open('./some/sample');
    childWins.push(win);
});
```

然后，当我们需要发送消息的时候，作为消息的发起方，一个页面需要同时通知它打开的页面与打开它的页面：

```js
// 过滤掉已经关闭的窗口
childWins = childWins.filter(w => !w.closed);
if (childWins.length > 0) {
    mydata.fromOpenner = false;
    childWins.forEach(w => w.postMessage(mydata));
}
if (window.opener && !window.opener.closed) {
    mydata.fromOpenner = true;
    window.opener.postMessage(mydata);
}
```

这样，作为消息发送方的任务就完成了。下面看看，作为消息接收方，它需要做什么。

此时，一个收到消息的页面就不能那么自私了，除了展示收到的消息，它还需要将消息再传递给它所“知道的人”（打开与被它打开的页面）:

> 需要注意的是，我这里通过判断消息来源，避免将消息回传给发送方，防止消息在两者间死循环的传递。（该方案会有些其他小问题，实际中可以进一步优化）

```js
window.addEventListener('message', function (e) {
    const data = e.data;
    const text = '[receive] ' + data.msg + ' —— tab ' + data.from;
    console.log('[Cross-document Messaging] receive message:', text);
    // 避免消息回传
    if (window.opener && !window.opener.closed && data.fromOpenner) {
        window.opener.postMessage(data);
    }
    // 过滤掉已经关闭的窗口
    childWins = childWins.filter(w => !w.closed);
    // 避免消息回传
    if (childWins && !data.fromOpenner) {
        childWins.forEach(w => w.postMessage(data));
    }
});
```

这样，每个节点（页面）都肩负起了传递消息的责任，也就是我说的“口口相传”，而消息就在这个**树状**结构中流转了起来。

> 显然，“口口相传”的模式存在一个问题：如果页面不是通过在另一个页面内的`window.open`打开的（例如直接在地址栏输入，或从其他网站链接过来），这个联系就被打破了。

### 非同源页面之间的通信

上面我们介绍了七种前端跨页面通信的方法，但它们大都受到同源策略的限制。然而有时候，我们有两个不同域名的产品线，也希望它们下面的所有页面之间能无障碍地通信。那该怎么办呢？

要实现该功能，可以使用一个用户不可见的 iframe 作为“桥”。由于 iframe 与父页面间可以通过指定`origin`来忽略同源限制，因此可以在每个页面中嵌入一个 iframe （例如：`http://sample.com/bridge.html`），而这些 iframe 由于使用的是一个 url，因此**属于同源**页面，其通信方式可以复用上面第一部分提到的各种方式。

页面与 iframe 通信非常简单，首先需要在页面中监听 iframe 发来的消息，做相应的业务处理：


```js
/* 业务页面代码 */
window.addEventListener('message', function (e) {
    // …… do something
});
```

然后，当页面要与其他的同源或非同源页面通信时，会先给 iframe 发送消息：

```js
/* 业务页面代码 */
window.frames[0].window.postMessage(mydata, '*');
```

其中为了简便此处将`postMessage`的第二个参数设为了`'*'`，你也可以设为 iframe 的 URL。iframe 收到消息后，会使用某种跨页面消息通信技术在所有 iframe 间同步消息，例如下面使用的 Broadcast Channel：

```js
/* iframe 内代码 */
const bc = new BroadcastChannel('AlienZHOU');
// 收到来自页面的消息后，在 iframe 间进行广播
window.addEventListener('message', function (e) {
    bc.postMessage(e.data);
});    
```

其他 iframe 收到通知后，则会将该消息同步给所属的页面：

```js
/* iframe 内代码 */
// 对于收到的（iframe）广播消息，通知给所属的业务页面
bc.onmessage = function (e) {
    window.parent.postMessage(e.data, '*');
};
```

下图就是使用 iframe 作为“桥”的非同源页面间通信模式图。



![img](/images/169d468988a6ba8ftplv-t2oaga2asx-zoom-in-crop-mark3024000.awebp)



其中“同源跨域通信方案”可以使用文章第一部分提到的某种技术。



## iframe通信

### iframe



### iframe跨域通信

https://juejin.cn/post/6844903831973675015#heading-3

#### 设置domain

`document.domain`作用是获取/设置当前文档的原始域部分，同源策略会判断两个文档的原始域是否相同来判断是否跨域。这意味着只要把这个值设置成一样就可以解决跨域问题了。

但是这个值的设置也有一定限制，只能设置为当前文档的上一级域或者是跟该文档的URL的domain一致的值。如url为a.demo.com，那domain就只能设置为demo.com或者a.demo.com。

因此，设置domain的方法只能用于解决**主域相同而子域不同**的情况。

#### postMessage

```js
otherWindow.postMessage(message, targetOrigin, [transfer]);
```

`otherWindow`

其他窗口的一个引用，比如iframe的contentWindow属性、执行[window.open](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fen-US%2Fdocs%2FWeb%2FAPI%2FWindow%2Fopen)返回的窗口对象、或者是命名过或数值索引的[window.frames](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fen-US%2Fdocs%2FWeb%2FAPI%2FWindow%2Fframes)。

`message`

将要发送到其他 window的数据。它将会被[结构化克隆算法](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fen-US%2Fdocs%2FDOM%2FThe_structured_clone_algorithm)序列化。这意味着你可以不受什么限制的将数据对象安全的传送给目标窗口而无需自己序列化。[[1](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fen-US%2Fdocs%2F)]

`targetOrigin`

通过窗口的origin属性来指定哪些窗口能接收到消息事件，其值可以是字符串"*"（表示无限制）或者一个URI。在发送消息的时候，如果目标窗口的协议、主机地址或端口这三者的任意一项不匹配targetOrigin提供的值，那么消息就不会被发送；只有三者完全匹配，消息才会被发送。这个机制用来控制消息可以发送到哪些窗口；例如，当用postMessage传送密码时，这个参数就显得尤为重要，必须保证它的值与这条包含密码的信息的预期接受者的origin属性完全一致，来防止密码被恶意的第三方截获。**如果你明确的知道消息应该发送到哪个窗口，那么请始终提供一个有确切值的targetOrigin，而不是\*。不提供确切的目标将导致数据泄露到任何对数据感兴趣的恶意站点。**

`transfer`

是一串和message 同时传递的 [`Transferable`](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fzh-CN%2Fdocs%2FWeb%2FAPI%2FTransferable) 对象. 这些对象的所有权将被转移给消息的接收方，而发送一方将不再保有所有权。

> 白话概括一下：
>
> - `otherWindow`就是你想要哪个页面的接受消息，你就用哪个页面的window来发送消息
> - `message`就是你想发送的数据
> - `targetOrigin`通常来讲就是otherWindow所在的**源**，协议+域名+端口需要一致，才能接受到消息
> - transfer一般忽略



下面看一下具体的例子

##### 父页面向子页面传递数据

**parent.html**

父页面通过`postMessage`向子页面通信，将`targetOrigin`设置为'*'，不做同源限制。

```html
<html>
  <head>
    <meta charset="utf-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>iframe</title>
  </head>
  <body>
    <div class="wrap">
      <div class="mb20 font-lg">Iframe</div>
      <iframe id="myIframe" src="http://127.0.0.1:3002/child/"></iframe>
    </div>
  </body>
  <script>
    const myIframe = document.getElementById('myIframe');
    myIframe.onload = function () {
      myIframe.contentWindow.postMessage('父页面发送的消息: 子页面url为http://127.0.0.1:3002/child/', '*');
    };
  </script>
</html>
```

**child.html**

子页面使用`window.addEventListener`监听父页面`message`事件，回调事件可以从`event.data`中获取到父页面传递的data。

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>iframe-child</title>
  </head>
  <body>
    <div id="child">
      <div style="margin-bottom: 10px;">子页面展示:</div>
    </div>
    <script>
      // 监听父页面message事件
      window.addEventListener('message', receive, false);
      // 回调函数
      function receive(event) {
        // 传递的data可以从event.data中获取到
        var div = document.querySelector('div');
        var node = document.createElement('div');
        node.innerHTML = event.data;
        div.appendChild(node);
      }
    </script>
  </body>
</html>

```



##### **子页面向父页面传递数据**

子页面通过也可以通过`postMessage`向父页面传递数据，不同的是otherWindow为父页面窗口，所以子页面需要使用`window.parent`表示父页面窗口

```js
window.parent.postMessage('子页面发送的消息', '*');
```

父页面使用`window.addEventListener`监听子页面`message`事件，回调事件可以从`event.data`中获取到子页面传递的data。

```js
// 监听子页面message事件
window.addEventListener('message', receive, false);
// 回调函数
function receive(event) {
    // 传递的data可以从event.data中获取到
    console.log(event.data);
}
```

