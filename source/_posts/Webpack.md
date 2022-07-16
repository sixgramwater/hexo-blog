---
title: webpack知识点总结
date: {{ date }}
tags: webpack
cover: https://ppoffice.github.io/hexo-theme-icarus/gallery/covers/vector_landscape_3.svg
excerpt: 这篇文章主要针对几个webpack零碎知识的总结：tree shaking, scope hoisting, code splitting, dynamic import，模块化....
toc: true
---
# Webpack

## 1. Tree shaking

### 原理

利用了ES6代码import的特点：

1. 只能作为模块顶层的语句出现

2. import的模块名只能是字符串常量
3. import binding 是immutable的

由于ES6模块加载是**静态**的（此处比较CJS的区别），因此，整个依赖树可以被静态地推导出解析语法树。使得可以在代码不运行阶段，就能分析出不需要的代码。

总结：

- `ES6 Module`引入进行静态分析，故而编译的时候正确判断到底加载了那些模块
- 静态分析程序流，判断那些模块和变量未被使用或者引用，进而删除对应代码

开启方式：在production mode自动开启

### cjs与es6模块引入的区别

`CommonJS` 是一种模块规范，最初被应用于 `Nodejs`，成为 `Nodejs` 的模块规范。运行在浏览器端的 `JavaScript` 由于也缺少类似的规范，在 `ES6` 出来之前，前端也实现了一套相同的模块规范 (例如: `AMD`)，用来对前端模块进行管理。自 `ES6` 起，引入了一套新的 `ES6 Module` 规范，在语言标准的层面上实现了模块功能，而且实现得相当简单，有望成为浏览器和服务器通用的模块解决方案。但目前浏览器对 `ES6 Module` 兼容还不太好，我们平时在 `Webpack` 中使用的 `export` 和 `import`，会经过 `Babel` 转换为 `CommonJS` 规范。在使用上的差别主要有：

1、`CommonJS` 模块输出的是一个值的拷贝，`ES6` 模块输出的是值的引用。

2、`CommonJS` 模块是运行时加载，`ES6` 模块是编译时输出接口。

3、`CommonJs` 是单个值导出，`ES6 Module`可以导出多个

4、`CommonJs` 是动态语法可以写在判断里，`ES6 Module` 静态语法只能写在顶层

5、`CommonJs` 的 `this` 是当前模块，`ES6 Module`的 `this` 是 `undefined`

参考：https://juejin.cn/post/6844904097556987917

## 2. Scope Hoisting

### 什么是 Scope Hoisting

默认情况下，经过 Webpack 打包后的模块资源会被组织成一个个函数形式，例如：

```js
// common.js 
export default "common"; 
 
// index.js 
import common from './common'; 
console.log(common); 
```

上例最终会被打包出形如下面结构的产物：

```js
"./src/common.js": 
  ((__unused_webpack_module, __webpack_exports__, __webpack_require__) => { 
     const __WEBPACK_DEFAULT_EXPORT__ = ("common"); 
     __webpack_require__.d(__webpack_exports__, { 
      /* harmony export */ 
      "default": () => (__WEBPACK_DEFAULT_EXPORT__) 
      /* harmony export */ 
    }); 
  }), 
"./src/index.js": 
  ((__unused_webpack_module, __webpack_exports__, __webpack_require__) => { 
      var _common__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__( /*! ./common */ "./src/common.js"); 
      console.log(_common__WEBPACK_IMPORTED_MODULE_0__) 
  }) 
```

每个模块都被重复的函数模板给包裹。

这种结构存在两个影响到运行性能的问题：

- 重复的函数模板代码会增大产物体积，消耗更多网络流量
- 函数的出栈入栈需要创建、销毁作用域空间，影响运行性能

针对这些问题，自 Webpack 3 开始引入 Scope Hoisting 功能，本质上就是将符合条件的多个模块合并到同一个函数空间内，减少函数声明的模板代码与运行时频繁出入栈操作，从而打包出「体积更小」、「运行性能」更好的包。例如上述示例经过 Scope Hoisting 优化后，生成代码：

```js
((__unused_webpack_module, __webpack_exports__, __webpack_require__) => { 
    ;// CONCATENATED MODULE: ./src/common.js 
    /* harmony default export */ const common = ("common"); 
     
    ;// CONCATENATED MODULE: ./src/index.js 
    console.log(common); 
}) 
```

### 原理总结

将所有（尽可能多的）模块的代码按照引用顺序放在一个函数作用域内，然后适当的重命名一些变量以防止变量名冲突。

正常来说 webpack 的引入都是把各个模块分开，通过 `__webpack_require__` 导入导出模块（对原理不熟悉的话可以看[这里](https://segmentfault.com/2019-02-19-webpack-bootstrap/)），但是使用 scope hoisting 后会把需要导入的文件直接移入导入者顶部

把每个模块被webpack处理成的模块初始化函数整理到一个统一的包裹函数里，也就是把**多个函数作用域**用一个作用域取代，以减少内存消耗并减少包裹块代码，从每个模块有一个包裹函数变成只有一个包裹函数包裹所有的模块，但是有一个前提就是，当模块的**引用次数大于1**时，比如被引用了两次或以上，那么这个效果会无效，也就是被引用多次的模块在被webpack处理后，会被独立的包裹函数所包裹

优点：减少函数声明代码和内存开销。

### 使用 Scope Hoisting

Webpack 提供了三种方法开启 Scope Hoisting 功能的方法：

- 开启 Production 模式
- 使用 optimization.concatenateModules 配置项
- 直接使用 ModuleConcatenationPlugin 插件

### 模块合并规则

开启 Scope Hoisting 后，Webpack 会将尽可能多的模块合并到同一个函数作用域下，但合并功能一方面依赖于 ESM 静态分析能力;一方面需要确保合并操作不会造成代码冗余。因此开发者需要注意 Scope Hoisting 会在以下场景下失效：

1. **非ESM模块**：对于 AMD、CMD 一类的模块，由于模块导入导出内容的动态性，Webpack 无法确保模块合并后不会对原有的代码语义产生副作用，导致 Scope Hoisting 失效
2. **模块被多个 Chunk 引用**：如果一个模块被多个 Chunk 同时引用，为避免重复打包，Scope Hoisting 同样会失效，该模块会被独立的包裹函数所包裹

## 3. code split

### splitchunks配置

```js
module.exports = {
  //...
  optimization: {
    splitChunks: {
      //在cacheGroups外层的属性设定适用于所有缓存组，不过每个缓存组内部可以重设这些属性
      chunks: "async", //将什么类型的代码块用于分割，三选一： "initial"：入口代码块 | "all"：全部 | "async"：按需加载的代码块
      minSize: 30000, //大小超过30kb的模块才会被提取
      maxSize: 0, //只是提示，可以被违反，会尽量将chunk分的比maxSize小，当设为0代表能分则分，分不了不会强制
      minChunks: 1, //某个模块至少被多少代码块引用，才会被提取成新的chunk
      maxAsyncRequests: 5, //分割后，按需加载的代码块最多允许的并行请求数，在webpack5里默认值变为6
      maxInitialRequests: 3, //分割后，入口代码块最多允许的并行请求数，在webpack5里默认值变为4
      automaticNameDelimiter: "~", //代码块命名分割符
      name: true, //每个缓存组打包得到的代码块的名称
      cacheGroups: {
        vendors: {
          test: /[\\/]node_modules[\\/]/, //匹配node_modules中的模块
          priority: -10, //优先级，当模块同时命中多个缓存组的规则时，分配到优先级高的缓存组
        },
        default: {
          minChunks: 2, //覆盖外层的全局属性
          priority: -20,
          reuseExistingChunk: true, //是否复用已经从原代码块中分割出来的模块
        },
      },
    },
  },
};

```

其中五个属性是控制代码分割规则的关键

- minSize(默认 30000)：使得比这个值大的模块才会被提取。
- minChunks（默认 1）：用于界定至少重复多少次的模块才会被提取。
- maxInitialRequests（默认 3）：一个代码块最终就会对应一个请求数，所以该属性决定入口最多分成的代码块数量，太小的值会使你无论怎么分割，都无法让入口的代码块变小。
- maxAsyncRequests（默认 5）：同上，决定每次按需加载时，代码块的最大数量。
- test：通过正则表达式精准匹配要提取的模块，可以根据项目结构制定各种规则，是手动优化的关键。

这些规则一旦制定，只有全部满足的模块才会被提取

## 4. HMR

## 5. 构建原理

## 6. 模块化原理

### CJS模块化

https://segmentfault.com/a/1190000010349749

webpack打包的代码，整体可以简化成下面的结构：

```javascript
(function (modules) {/* 省略函数内容 */})
([
function (module, exports, __webpack_require__) {
    /* 模块index.js的代码 */
},
function (module, exports, __webpack_require__) {
    /* 模块bar.js的代码 */
}
]);
```

可以看到，整个打包生成的代码是一个IIFE(立即执行函数)，函数内容我们待会看，我们先来分析函数的参数。

下面是摘取的函数内容，并添加了一些注释：

```java
// 1、模块缓存对象
var installedModules = {};
// 2、webpack实现的require
function __webpack_require__(moduleId) {
    // 3、判断是否已缓存模块
    if(installedModules[moduleId]) {
        return installedModules[moduleId].exports;
    }
    // 4、缓存模块
    var module = installedModules[moduleId] = {
        i: moduleId,
        l: false,
        exports: {}
    };
    // 5、调用模块函数
    modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
    // 6、标记模块为已加载
    module.l = true;
    // 7、返回module.exports
    return module.exports;
}
// 8、require第一个模块
return __webpack_require__(__webpack_require__.s = 0);
```

模块数组作为参数传入IIFE函数后，IIFE做了一些初始化工作：

1. IIFE首先定义了`installedModules `，这个变量被用来**缓存**已加载的模块。
2. 定义了`__webpack_require__ `这个函数，函数参数为模块的id。这个函数用来实现模块的require。
3. `__webpack_require__ `函数首先会检查是否缓存了已加载的模块，如果有则直接返回缓存模块的`exports`。
4. 如果没有缓存，也就是第一次加载，则首先初始化模块，并将模块进行缓存。
5. 然后调用模块函数，也就是前面webpack对我们的模块的包装函数，将`module`、`module.exports`和`__webpack_require__`作为参数传入。注意这里做了一个动态绑定，将模块函数的调用对象绑定为`module.exports`，这是为了保证在模块中的this指向当前模块。
6. 调用完成后，模块标记为已加载。
7. 返回模块`exports`的内容。
8. 利用前面定义的`__webpack_require__ `函数，require第0个模块，也就是入口模块。

require入口模块时，入口模块会收到收到三个参数，下面是入口模块代码：

```javascript
function(module, exports, __webpack_require__) {
    "use strict";
    var bar = __webpack_require__(1);
    bar.bar();
}
```

webpack传入的第一个参数`module`是当前缓存的模块，包含当前模块的信息和`exports`；第二个参数`exports`是`module.exports`的引用，这也符合commonjs的规范；第三个`__webpack_require__ `则是`require`的实现。

在我们的模块中，就可以对外使用`module.exports`或`exports`进行导出，使用`__webpack_require__`导入需要的模块，代码跟commonjs完全一样。

这样，就完成了对第一个模块的require，然后第一个模块会根据自己对其他模块的require，依次加载其他模块，最终形成一个依赖网状结构。webpack管理着这些模块的缓存，如果一个模块被require多次，那么只会有一次加载过程，而返回的是缓存的内容，这也是commonjs的规范。



### ESM模块化

https://segmentfault.com/a/1190000010955254

我们依然写两个文件，m.js文件用es模块的方式`export`一个`default`函数和一个`foo`函数，index.js `import`该模块，具体代码如下：

```javascript
// m.js
'use strict';
export default function bar () {
    return 1;
};
export function foo () {
    return 2;
}
// index.js
'use strict';
import bar, {foo} from './m';
bar();
foo();
```

webpack配置没有变化，依然以index.js作为入口：

```lua
var path = require("path");
module.exports = {
    entry: path.join(__dirname, 'index.js'),
    output: {
        path: path.join(__dirname, 'outs'),
        filename: 'index.js'
    },
};
```

在根目录下执行`webpack`，得到经过webpack打包的代码如下（去掉了不必要的注释）：

```javascript
(function(modules) { // webpackBootstrap
    // The module cache
    var installedModules = {};
    // The require function
    function __webpack_require__(moduleId) {
        // Check if module is in cache
        if(installedModules[moduleId]) {
            return installedModules[moduleId].exports;
        }
        // Create a new module (and put it into the cache)
        var module = installedModules[moduleId] = {
            i: moduleId,
            l: false,
            exports: {}
        };
        // Execute the module function
        modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
        // Flag the module as loaded
        module.l = true;
        // Return the exports of the module
        return module.exports;
    }
    // expose the modules object (__webpack_modules__)
    __webpack_require__.m = modules;
    // expose the module cache
    __webpack_require__.c = installedModules;
    // define getter function for harmony exports
    __webpack_require__.d = function(exports, name, getter) {
        if(!__webpack_require__.o(exports, name)) {
            Object.defineProperty(exports, name, {
                configurable: false,
                enumerable: true,
                get: getter
            });
        }
    };
    // getDefaultExport function for compatibility with non-harmony modules
    __webpack_require__.n = function(module) {
        var getter = module && module.__esModule ?
            function getDefault() { return module['default']; } :
            function getModuleExports() { return module; };
        __webpack_require__.d(getter, 'a', getter);
        return getter;
    };
    // Object.prototype.hasOwnProperty.call
    __webpack_require__.o = function(object, property) { return Object.prototype.hasOwnProperty.call(object, property); };
    // __webpack_public_path__
    __webpack_require__.p = "";
    // Load entry module and return exports
    return __webpack_require__(__webpack_require__.s = 0);
})
([
(function(module, __webpack_exports__, __webpack_require__) {

    "use strict";
    Object.defineProperty(__webpack_exports__, "__esModule", { value: true });
    /* harmony import */
    var __WEBPACK_IMPORTED_MODULE_0__m__ = __webpack_require__(1);

    Object(__WEBPACK_IMPORTED_MODULE_0__m__["a" /* default */])();
    Object(__WEBPACK_IMPORTED_MODULE_0__m__["b" /* foo */])();

}),
(function(module, __webpack_exports__, __webpack_require__) {

    "use strict";
    /* harmony export (immutable) */
    __webpack_exports__["a"] = bar;
    /* harmony export (immutable) */
    __webpack_exports__["b"] = foo;

    function bar () {
        return 1;
    };
    function foo () {
        return 2;
    }

})
]);
```

#### 分析

上一篇文章已经分析过了，webpack生成的代码是一个IIFE，这个IIFE完成一系列初始化工作后，就会通过`__webpack_require__(0)`启动入口模块。

我们首先来看m.js模块是如何实现es的`export`的，被webpack转换后的m.js代码如下：

```js
__webpack_exports__["a"] = bar;
__webpack_exports__["b"] = foo;

function bar () {
    return 1;
};
function foo () {
    return 2;
}
```

其实一眼就能看出来，export default和export都被转换成了类似于commonjs的`exports.xxx`，这里也已经不区分是不是default export了，所有的export对象都是`__webpack_exports__`的属性。

我们继续来看看入口模块，被webpack转换后的index.js代码如下：

```js
Object.defineProperty(__webpack_exports__, "__esModule", { value: true });
var __WEBPACK_IMPORTED_MODULE_0__module__ = __webpack_require__(1);

Object(__WEBPACK_IMPORTED_MODULE_0__m__["a" /* default */])();
Object(__WEBPACK_IMPORTED_MODULE_0__m__["b" /* foo */])();
```

index模块首先通过`Object.defineProperty`在`__webpack_exports__`上添加属性`__esModule `，值为`true`，表明这是一个es模块。在目前的代码下，这个标记是没有作用的，至于在什么情况下需要判断模块是否es模块，后面会分析。

然后就是通过`__webpack_require__(1)`导入m.js模块，再然后通过`module.xxx`获取m.js中export的对应属性。注意这里有一个重要的点，就是所有引入的模块属性都会用Object()包装成对象，这是为了保证像Boolean、String、Number这些基本数据类型转换成相应的类型对象。

#### commonjs与es6 module混用

#### 结论

webpack对于es模块的实现，也是基于自己实现的`__webpack_require__ `和`__webpack_exports__ `，装换成类似于commonjs的形式。对于es模块和commonjs混用的情况，则需要通过`__webpack_require__.n`的形式做一层包装来实现。

下一篇[webpack模块化原理-Code Splitting](https://segmentfault.com/a/1190000011435407)，会继续来分析webpack是如何通过动态`import`和`module.ensure`实现Code Splitting的。

### webpack实现code splitting(动态加载)

webpack的模块化不仅支持commonjs和es module，还能通过code splitting实现模块的动态加载。根据[wepack官方文档](https://link.segmentfault.com/?enc=AK%2B9q89hP7mSCCfAFISiRw%3D%3D.cNX%2B5SJSD6n6vSb%2FK%2FdatFlKWRXPNIUyQ5rgcMdLAqULBl0x8v1BA12EAju%2FqFv3vUB5djWXmb9DGX7AlongLqV3MvjtwB0mdcGr81cdPwVnaFc%2BrwdN6fLA6zj8EDox)，实现动态加载的方式有两种：`import`和`require.ensure`。

那么，这篇文档就来分析一下，webpack是如何实现code splitting的。

首先我们依然创建一个简单入口模块`index.js`和两个依赖模块`foo.js`和`bar.js`：

```javascript
// index.js
'use strict';
import(/* webpackChunkName: "foo" */ './foo').then(foo => {
    console.log(foo());
})
import(/* webpackChunkName: "bar" */ './bar').then(bar => {
    console.log(bar());
})
// foo.js
'use strict';
exports.foo = function () {
    return 2;
}
// bar.js
'use strict';
exports.bar = function () {
    return 1;
}
```

webpack配置如下：

```lua
var path = require("path");

module.exports = {
    entry: path.join(__dirname, 'index.js'),
    output: {
        path: path.join(__dirname, 'outs'),
        filename: 'index.js',
        chunkFilename: '[name].bundle.js'
    },
};
```

这是一个最简单的配置，指定了模块入口和打包文件输出路径，值得注意的是，这次还指定了分离模块的文件名`[name].bundle.js`（不指定会有默认文件名）。

#### 分析

编译后的代码，整体跟前两篇文章中使用commonjs和es6 module编写的代码编译后的结构差别不大，都是通过IFFE的方式启动代码，然后使用webpack实现的`require`和`exports`实现的模块化。

而对于code splitting的支持，区别在于这里使用`__webpack_require__.e`实现动态加载模块和实现基于promise的模块导入。

所以首先分析`__webpack_require__.e`函数的定义，这个函数实现了动态加载：

```javascript
__webpack_require__.e = function requireEnsure(chunkId) {
    // 1、缓存查找
    var installedChunkData = installedChunks[chunkId];
    if(installedChunkData === 0) {
        return new Promise(function(resolve) { resolve(); });
    }
    if(installedChunkData) {
        return installedChunkData[2];
    }
    // 2、缓存模块
    var promise = new Promise(function(resolve, reject) {
        installedChunkData = installedChunks[chunkId] = [resolve, reject];
    });
    installedChunkData[2] = promise;
    // 3、加载模块
    var head = document.getElementsByTagName('head')[0];
    var script = document.createElement('script');
    script.type = 'text/javascript';
    script.charset = 'utf-8';
    script.async = true;
    script.timeout = 120000;
    if (__webpack_require__.nc) {
        script.setAttribute("nonce", __webpack_require__.nc);
    }
    script.src = __webpack_require__.p + "" + ({"0":"foo"}[chunkId]||chunkId) + ".bundle.js";
    // 4、异常处理
    var timeout = setTimeout(onScriptComplete, 120000);
    script.onerror = script.onload = onScriptComplete;
    function onScriptComplete() {
        // avoid mem leaks in IE.
        script.onerror = script.onload = null;
        clearTimeout(timeout);
        var chunk = installedChunks[chunkId];
        if(chunk !== 0) {
            if(chunk) {
                chunk[1](new Error('Loading chunk ' + chunkId + ' failed.'));
            }
            installedChunks[chunkId] = undefined;
        }
    };
    head.appendChild(script);
    // 5、返回promise
    return promise;
};
```

代码大致逻辑如下：

1. 缓存查找：从缓存`installedChunks`中查找是否有缓存模块，如果缓存标识为0，则表示模块已加载过，直接返回`promise`（状态为resolved）；如果缓存为数组，表示缓存正在加载中，则返回缓存的`promise`对象
2. 如果没有缓存，则创建一个`promise`，并将`promise`和`resolve`、`reject`缓存在`installedChunks`中
3. **构建一个script标签**，append到head标签中，src指向加载的模块脚本资源，实现动态加载js脚本
4. 添加script标签onload、onerror 事件，如果超时或者模块加载失败，则会调用reject返回模块加载失败异常
5. 如果模块加载成功，则返回当前模块`promise`，对应于`import()`

以上便是模块加载的过程，当资源加载完成，模块代码开始执行，那么我们来看一下（动态加载的）模块代码的结构：

```javascript
webpackJsonp([0],[
/* 0 */,
/* 1 */
/***/ (function(module, exports, __webpack_require__) {
"use strict";
exports.foo = function () {
    return 2;
}
/***/ })
]);
```

可以看到，模块代码不仅被包在一个函数中（用来模拟模块作用域），外层还被当做参数传入`webpackJsonp`中。那么这个`webpackJsonp`函数的作用是什么呢？

其实这里的`webpackJsonp`类似于jsonp中的callback，作用是**作为模块加载和执行完成的回调**，从而触发`import`的`resolve`。

具体细看`webpackJsonp`代码来分析：

```js
window["webpackJsonp"] = function webpackJsonpCallback(chunkIds, moreModules, executeModules) {
    var moduleId, chunkId, i = 0, resolves = [], result;
    // 1、收集模块resolve
    for(;i < chunkIds.length; i++) {
        chunkId = chunkIds[i];
        if(installedChunks[chunkId]) {
            resolves.push(installedChunks[chunkId][0]);
        }
        installedChunks[chunkId] = 0;
    }
    // 2、copy模块到modules
    for(moduleId in moreModules) {
        if(Object.prototype.hasOwnProperty.call(moreModules, moduleId)) {
            modules[moduleId] = moreModules[moduleId];
        }
    }
    if(parentJsonpFunction) parentJsonpFunction(chunkIds, moreModules, executeModules);
    // 3、resolve import
    while(resolves.length) {
        resolves.shift()(); // 调用resolve()
    }
};
```

代码大致逻辑如下：

1. 根据`chunkIds`收集对应模块的`resolve`，这里的`chunkIds`为数组是因为`require.ensure`是可以实现异步加载多个模块的，所以需要兼容
2. 把动态模块添加到IFFE的`modules`中，提供其他CMD方案使用模块
3. 直接调用`resolve`，完成整个异步加载

#### 总结

webpack通过`__webpack_require__.e`函数实现了动态加载，再通过`webpackJsonp`函数实现异步加载回调，把模块内容以promise的方式暴露给调用方，从而实现了对code splitting的支持。
