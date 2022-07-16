## NPM包管理

npm的包管理机制你一定要了解，不仅仅是node需要，我们前端浏览器项目本身也会引用很多第三方模块。面试必备知识点。

下图摘自抖音前端团队的[npm包管理机制](https://juejin.cn/post/6844904022080667661#heading-32) [![img](https://camo.githubusercontent.com/6f3c50959e9aeef5ec2ddf386bf41bbdc3f7497737d729e095fde592702662be/68747470733a2f2f70312d6a6a2e62797465696d672e636f6d2f746f732d636e2d692d74326f616761326173782f676f6c642d757365722d6173736574732f323031392f31322f31362f313666306565663332376363616261357e74706c762d74326f616761326173782d7a6f6f6d2d696e2d63726f702d6d61726b3a313330343a303a303a302e6177656270)](https://camo.githubusercontent.com/6f3c50959e9aeef5ec2ddf386bf41bbdc3f7497737d729e095fde592702662be/68747470733a2f2f70312d6a6a2e62797465696d672e636f6d2f746f732d636e2d692d74326f616761326173782f676f6c642d757365722d6173736574732f323031392f31322f31362f313666306565663332376363616261357e74706c762d74326f616761326173782d7a6f6f6d2d696e2d63726f702d6d61726b3a313330343a303a303a302e6177656270)

本图如果你理解的话，后面的内容就不用看了。

讲npm install 要从嵌套结构讲起

### 嵌套结构

在 npm 的早期版本中，npm 处理依赖的方式简单粗暴，以递归的方式，严格按照 package.json 结构以及子依赖包的 package.json 结构将依赖安装到他们各自的 node_modules 中。

如下图： [<img src="https://camo.githubusercontent.com/166c5e1672bcef3c7d14c58d0a2c0a95964e84f95244d62b1b03fb0ec29f8f99/68747470733a2f2f7778736d2e73706163652f323032312f6e706d2d686973746f72792f62643936663130323234346334613439386335343462663735393236366538312e706e67" alt="img" style="zoom: 67%;" />](https://camo.githubusercontent.com/166c5e1672bcef3c7d14c58d0a2c0a95964e84f95244d62b1b03fb0ec29f8f99/68747470733a2f2f7778736d2e73706163652f323032312f6e706d2d686973746f72792f62643936663130323234346334613439386335343462663735393236366538312e706e67) 

这样的方式优点很明显， node_modules 的结构和 package.json 结构一一对应，层级结构明显，并且保证了每次安装目录结构都是相同的。

从上图这种情况，我们不难得出嵌套结构拥有以下缺点：

- 在不同层级的依赖中，可能引用了同一个模块，导致大量冗余
- 在 Windows 系统中，文件路径最大长度为260个字符，嵌套层级过深可能导致不可预知的问题

### 扁平结构

2016 年，yarn 诞生了。yarn 解决了 npm 几个最为迫在眉睫的问题：

- 安装太慢（加缓存、多线程）
- 嵌套结构（扁平化）
- 无依赖锁（yarn.lock）
- yarn 带来对的扁平化结构：

如下图，我们简单看下什么是扁平化的结构：

没错，这就是扁平化依赖管理的结果。相比之前的嵌套结构，现在的目录结构类似下面这样: 假如之前嵌套的结构如下：

```
node_modules
├─ a
|  ├─ index.js
|  |- node_modules -└─ b
|  |                ├─ index.js
|  |                └─ package.json
|  └─ package.json
```

那么扁平化处理以后，就编程下面这样，被拍平了

```
node_modules
├─ a
|  ├─ index.js
|  └─ package.json
└─ b
   ├─ index.js
   └─ package.json
```

但是扁平化的结构又会引出新的问题：

最主要的就是**依赖结构的不确定性**！

啥意思，我就懒得画图了，拿网上的一个例子来说：

想象一下有一个 library-a，它同时依赖了 library-b、c、d、e：

[<img src="https://camo.githubusercontent.com/ee33198408055e307e46277ecf16892ea072e6767750ca03b3b0a8cf47cfd536/68747470733a2f2f7778736d2e73706163652f323032312f6e706d2d686973746f72792f31376434376564643438366234613637616436343632626163643161346339342e706e67" alt="img" style="zoom: 67%;" />](https://camo.githubusercontent.com/ee33198408055e307e46277ecf16892ea072e6767750ca03b3b0a8cf47cfd536/68747470733a2f2f7778736d2e73706163652f323032312f6e706d2d686973746f72792f31376434376564643438366234613637616436343632626163643161346339342e706e67)

而 b 和 c 依赖了 f@1.0.0，d 和 e 依赖了 f@2.0.0：

[<img src="https://camo.githubusercontent.com/69b7c0d5fd6abcf43483f12921278d9ce1168f6797efb331eeea4454adb78427/68747470733a2f2f7778736d2e73706163652f323032312f6e706d2d686973746f72792f39656239323937316362303434636233613035633632613433646432333134322e706e67" alt="img" style="zoom: 67%;" />](https://camo.githubusercontent.com/69b7c0d5fd6abcf43483f12921278d9ce1168f6797efb331eeea4454adb78427/68747470733a2f2f7778736d2e73706163652f323032312f6e706d2d686973746f72792f39656239323937316362303434636233613035633632613433646432333134322e706e67)

这时候，node_modules 树需要做出选择了，到底是将 f@1.0.0 还是 f@2.0.0 扁平化，然后将另一个放到嵌套的 node_modules 中？

答案是：具体做那种选择将是不确定的，取决于哪一个 f 出现得更靠前，靠前的那个将被扁平化。

还有一个问题就是`幽灵依赖`，明明只安装a包，你却可以引用b包，因为a引用了b，并且扁平化处理了。

### lock文件

这就是为啥要有lock文件的原因，lock文件可以**保证安装包的扁平化结构的稳定**。

### pnpm如何解决上述问题？

pnpm? 可以简单介绍一下为啥它能解决上面扁平化结构和幽灵依赖的问题。

![img](\images\27a9da5df7214075b0f02e9b93b596f0tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp)

#### 软链接与硬链接

##### hard link

硬链接可以理解为是一个相互的指针，创建的 hardlink 指向源文件的 inode，系统并不为它重新分配 inode。

硬链接不管有多少个，都指向的是同一个 inode 节点，这意味着当你修改源文件或者链接文件的时候，都会做同步的修改。

每新建一个 hardlink 会把节点连接数增加，只要节点的链接数非零，文件就一直存在，不管你删除的是源文件还是 hradlink。只要有一个存在，文件就存在（类似引用计数的概念。

<img src="\images\7c88be23a0e4497da07f4877269c1b35tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp" alt="img" style="zoom:50%;" />\

##### soft link

软链接可以理解为是一个单向指针，是一个独立的文件且拥有独立的 inode，永远指向源文件，这就类比于 Windows 系统的快捷方式。

<img src="\images\27ffc9df1d3846efabac1cbd85bdf4ectplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp" alt="img" style="zoom:50%;" />




#### pnpm依赖管理

只有直接依赖的包会出现在node_modules下，如express。express自身所依赖的包会被打平，平铺在`.pnpm`文件夹

pnpm 是通过 hardlink 在全局里面搞个 store 目录来存储 node_modules 依赖里面的 hard link 地址，然后在引用依赖的时候则是通过 symlink 去找到对应虚拟磁盘目录下(.pnpm 目录)的依赖地址。

![node-modules-structure-8ab301ddaed3b7530858b233f5b3be57.jpeg](\images\aee0c772a3294709bef5b25fc9d8c89atplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp)

图中`ln`代表硬链接， `ln -s`代表软链接.	

#### hard link机制解决重复安装

硬链接的作用是允许一个文件拥有多个有效路径名，因此，即使我们在目录中存放多个hard link，只要它们指向相同的文件，那么它们永远都只会占用一份空间。

使用 pnpm 安装，pnpm 会将依赖存储在位于 `~/.pnpm-store` 目录下。只要你在同一机器下，下次安装依赖的时候 pnpm 会先检查 store 目录，如果有你需要的依赖则会通过一个硬链接丢到到你的项目中去，而不是重新安装依赖。



#### pnpm如何解决幽灵依赖的问题？

`node_modules`中并不是和yarn一样的扁平结构，而是仅存放到`.pnpm`目录相应位置的`软链接`，第一眼看过去`node_modules`中的结构和`package.json`中声明的依赖保持一致。

pnpm 这种依赖管理的方式也很巧妙地规避了`非法访问依赖`的问题，也就是只要一个包未在 package.json 中声明依赖，那么在项目中是无法访问的。



## 模块

https://segmentfault.com/a/1190000023828613

### CJS如何解决循环加载问题？(互相require)

#### ES6 import如何解决？



## 异步I/O

## Stream

https://juejin.cn/post/7077511716564631566#heading-0

## Koa原理

https://segmentfault.com/a/1190000037769323?utm_source=sf-similar-article

### koa用法
```js
const Koa = require('koa');
const app = new Koa();

app.use((ctx, next) => {
  ctx.body = 'Hello World';
});

app.listen(3000); 
```

原生方法
```js
let http = require('http')

let server = http.createServer((req, res) => {
  res.end('hello world')
})

server.listen(4000)
```
相对原生，koa多了两个实例上的`use`、`listen`方法，和use回调中的`ctx`、`next`两个参数。这四个不同，几乎就是koa的全部了，也是这四个不同让koa如此强大。

### 上下文ctx的实现
#### 绑定request，response对象
```js
// this.fn = ...

use(fn) {
	this.fn = fn;
}

createContext(req, res) {
  const ctx = Object.create(this.context);
  const request = (ctx.request = Object.create(this.request));
  const response = (ctx.response = Object.create(this.response));

  ctx.req = request.req = response.req = req;
  ctx.res = request.res = response.res = res;
  request.ctx = response.ctx = ctx;
  request.response = response;
  request.request = request;
  return ctx;
}

// 
handleRequest(req, res) {
  res.statusCode = 404; // 默认404
  let ctx = this.createContext(req, res);
  this.fn(ctx);
  
}

listen(...args) {
   let server = http.createServer(this.handleRequest.bind(this));
   server.listen(...args);
}
```

#### 代理ctx.response/ctx.request
目前我们只能通过ctx.request.query的方式来访问query。
那么我们如何实现只调用ctx.query、ctx.url，来访问ctx.request.query呢？

答案是使用代理,源码中使用__defineGetter__和__defineSetter__来创建对ctx.url的代理，将它代理到ctx.request.url上
```js
let proto = {};

// 使用代理将ctx.url代理到ctx.request.url
function defineGetter(prop, name) {
  // 创建一个defineGetter函数，参数分别是要代理的对象和对象上的属性
  proto.__defineGetter__(name, function () {
    // 每个对象都有一个__defineGetter__方法，可以用这个方法实现代理，下面详解
    return this[prop][name]; // 这里的this是ctx（原因下面解释），所以ctx.url得到的就是this.request.url
  });
}

function defineSetter(prop, name) {
  proto.__defineSetter__(name, function (val) {
    // 用__defineSetter__方法设置值
    this[prop][name] = val;
  });
}

defineGetter("request", "url"); // 代理response的url和path属性
defineGetter("request", "path");

defineGetter("response", "body"); // 同样代理response的body属性
defineSetter("response", "body"); // 同理
```

### 洋葱模型的实现
核心在于compose函数

#### 1. 让多个use的回调按照顺序排列成串
```js
class Koa extends EventEmitter {
  constructor() {
    super();

    this.middlewares = [];
    this.context = context;
    this.request = request;
    this.response = response;
  }
  use(fn) {
    this.middlewares.push(fn);
  }

  // 简化版的compose，接收中间件数组、ctx对象作为参数
  compose(middlewares, ctx) {
    function dispatch(index) {
      // 利用递归函数将各中间件串联起来依次调用
      if (index === middlewares.length) return; // 最后一次next不能执行，不然会报错
      let middleware = middlewares[index]; // 取当前应该被调用的函数
      middleware(ctx, () => dispatch(index + 1)); // 调用并传入ctx和下一个将被调用的函数，用户next()时执行该函数
    }
    dispatch(0);
  }

  createContext(req, res) {
    const ctx = Object.create(this.context);
    const request = (ctx.request = Object.create(this.request));
    const response = (ctx.response = Object.create(this.response));

    ctx.req = request.req = response.req = req;
    ctx.res = request.res = response.res = res;
    request.ctx = response.ctx = ctx;
    request.response = response;
    request.request = request;
    return ctx;
  }

  handleRequest(req, res) {
    res.statusCode = 404; // 默认404
    let ctx = this.createContext(req, res);
    // this.fn(ctx);
    this.compose(this.middlewares, ctx) // 调用compose，传入参数
    if (typeof ctx.body == "object") {
      // 如果是个对象，按json形式输出
      res.setHeader("Content-Type", "application/json;charset=utf8");
      res.end(JSON.stringify(ctx.body));
    } else if (ctx.body instanceof Stream) {
      // 如果是流
      ctx.body.pipe(res);
    } else if (typeof ctx.body === "string" || Buffer.isBuffer(ctx.body)) {
      // 如果是字符串或buffer
      res.setHeader("Content-Type", "text/html;charset=utf8");
      res.end(ctx.body);
    } else {
      res.end("Not found");
    }
  }

  listen(...args) {
    let server = http.createServer(this.handleRequest.bind(this));
    server.listen(...args);
  }
}
```

#### 2. 把每个回调包装成Promise以实现异步
```js
compose(middlewares, ctx) {
  function dispatch(index) {
    // 利用递归函数将各中间件串联起来依次调用
    if (index === middlewares.length) return Promise.resolve(); // 最后一次next不能执行，不然会报错
    let middleware = middlewares[index]; // 取当前应该被调用的函数
    return Promise.resolve(middleware(ctx, () => dispatch(index + 1))); // 用Promise.resolve把中间件包起来
  }
  return dispatch(0);
}


handleRequest() {
	let fn = this.compose(this.middlewares, ctx);
	fn.then(()=>{
    	//...
    })
}
```