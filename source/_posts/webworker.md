---
title: web worker
date: {{ date }}
tags: JS
cover: https://ppoffice.github.io/hexo-theme-icarus/gallery/covers/vector_landscape_1.svg
excerpt: 简单介绍了web worker以及使用场景、通信等知识。
toc: true
---
## Web Worker 介绍

众所周知，JavaScript 这门语言的一大特点就是单线程，即同一时间只能同步处理一件事情，这也是这门语言衍生出的 nodeJS 被各后端大佬诟病的很重要的一点。

然而，JavaScript 在设计之初，其实是故意被设计成单线程语言的，这是由于它当时的主要用途决定的。

JavaScript 最初的设计初衷是完成页面与用户的交互，操作 DOM 或者 BOM 元素，此时如果一味地追求效率使用多线程的话，会带来资源抢占，数据同步等等问题，因此必须规定，同一时间只有一个线程能直接操作页面元素，以保证系统的稳定性以及安全性。

尽管如此，但是 JavaScript 并不是只能线性处理任务。JS 拥有消息队列和事件循环机制，通过异步处理消息的能力来实现并发。在高 I/O 型并发事务处理的过程中，由于不需要手动生成与销毁线程以及占用额外管理线程的空间，性能表现及为优异。因此，nodeJS 作为 JavaScript 在服务端的探索者，在处理高并发网络请求的优势极为明显。

尽管 JavaScript 通过异步机制完美解决了高 I/O 性能的问题，但 JavaScript 单线程执行的本质还是没有变的。因此缺点就显而易见了，那就是**处理 CPU 密集型的事务时没有办法充分调动现代多核心多线程机器的运算资源**。

在现代大型前端项目中，随着代码的复杂程度越来越高，本地的**计算型事务**也在变得繁重，而运行在单线程下 JS 项目必定会忙于处理计算而无暇顾及用户接下来的频繁操作，造成卡顿等不太好的用户体验，更严重的情况是，当计算型事务过多时还有可能因为资源被占满带来网页无响应的卡死现象。因此，Web 项目的本地多线程运算能力势在必行，由此，Web Worker 应运而生了。

Web Worker 是 HTML5 中推出的标准，官方是这样定义它的：

> Web Workers makes it possible to run a script operation in a background thread separate from the main execution thread of a web application.

它允许 JavaScript 脚本创建多个线程，从而充分利用 CPU 的多核计算能力，不会阻塞主线程 (一般指 UI 渲染线程) 的运行。

Web Worker 虽然是 HTML5 标准，但其实早在 2009 年 W3C 就已经提出了草案，因此它的兼容性良好，基本覆盖了所有主流浏览器。

[![20200703113954](https://raw.githubusercontent.com/kelekexiao123/blog-storage/master/images/20200703113954.png)](https://raw.githubusercontent.com/kelekexiao123/blog-storage/master/images/20200703113954.png)

## Web Worker 的局限

需要注意的是，Web Worker 本质上并没有突破 JavaScript 的单线程的性质。

事实上，Web Worker 脚本中的代码并不能直接操作 DOM 节点，并且不能使用绝大多数 BOM API。它的全局环境是 DedicatedWorkerGlobalScope 而并不是 Window。运行 Worker 的实际上是一个沙箱，跑的是与主线程完全独立 JavaScript 文件。

Worker 做的这些限制，实际上也是为了避免文章开头说过的抢占问题。它更多的使用场景是作为主线程的附属，完成高 CPU 计算型的数据处理，再通过线程间通信将执行结果传回给主线程。在整个过程中，主线程仍然能正常地相应用户操作，从而很好地避免页面的卡顿现象。

## Web Worker 的使用

### 新建

目前 Web Worker 的浏览器支持已经较为完善，基本上直接传入 Worker 脚本的 URI 并实例化即可使用。

```js
/* main.js */

const worker = new Worker("./worker.js")
```



### 通信

Worker 与主线程之间的通信只需要各有两个 API：`onmessage`/`addEventListener` 与` postMessage `即可完成收发消息的交互。

```js
/* main.js */
const worker = new Worker("./worker.js");
 
// 主线程发送消息
worker.postMessage({ data: 'mainthread send data' });
 
// 主线程接收消息
worker.onmessage = (e) => {
    const { data } = e;
    if (!data) return;
    console.log(data);
}
```

```js
/* worker.js */
// worker线程接收消息
self.addEventListener('message', (e) => {
    const { data } = e;
    if (!data) return;
    // worker线程发送消息
    self.postMessage({data: 'worker received data'})
});
```

注：Worker 中，`this.xx`， `self.xx `与直接使用 xx，其作用域都指向 worker 的全局变量` DedicatedWorkerGlobalScope` ，可以互换。

### 销毁

Worker 的销毁方式有两种，既能在内部主动销毁，也能够被主线程通知销毁。

```js
/* main.js */
worker.terminate();
```

```js
/* worker.js */
self.close();
```

## 进阶：让通信方式 Promise 化

http://www.alloyteam.com/2020/07/14645/

## Web Worker的应用

