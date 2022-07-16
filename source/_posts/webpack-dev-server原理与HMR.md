---
title: webpack-dev-server与HMR原理
date: {{ date }}
tags: webpack
cover: https://ppoffice.github.io/hexo-theme-icarus/gallery/covers/vector_landscape_1.svg
excerpt: 这篇文章深入剖析了webpack-dev-server的原理，以及HMR原理。
toc: true
---
# webpack-dev-server原理与HMR

## 流程图

![](https://raw.githubusercontent.com/sixgramwater/markdown-imgs/master/20220303211351.png)

## 入口
如果作为命令行启动，`webpack-dev-server/bin/webpack-dev-server.js` 就是整个命令行的入口
```js
// webpack-dev-server/bin/webpack-dev-server.js
 
function startDevServer(config, options) {
 
  let compiler;
 
  try {
    // 2. 调用webpack函数返回的是 webpack compiler 实例
    compiler = webpack(config);
  } catch (err) {
  }
 
  try {
    // 3. 实例化 webpack-dev-server
    server = new Server(compiler, options, log);
  } catch (err) {
  }
 
  if (options.socket) {
  } else {
    // 4. 调用 server 实例的 listen 方法
    server.listen(options.port, options.host, (err) => {
      if (err) {
        throw err;
      }
    });
  }
}
 
// 1. 对参数进行处理后启动
processOptions(config, argv, (config, options) => {
  startDevServer(config, options);
});

```

首先是调用了 `webpack-cli` 模块下的两个文件，分别配置了命令行提示选项、和从命令行和配置文件收集了 webpack 的 config

之后调用 `processOptions` 对收集的参数进行一些默认处理后得到需要传给 webpack 的 config 和需要传给` wepack-dev-server` 的 `options`。

`startDevServer `这个函数主要是先调用 `webpack` 函数实例化了 `compiler`，注意这里没有给 webpack 函数传入回调函数，根据 webpack 源码实现，不传入回调函数就不会直接运行 webpack 而是返回webpack `compiler `的实例，供调用方自行启动 webpack 运行

最后调用 Server 类的 listen 方法，就正式开启监听请求

## Server
```js
// webpack-dev-server/lib/Server.js
 
class Server {
  constructor(compiler, options = {}, _log) {
    // 0. 校验参数是否符合 schema, 不符合会抛出错误
    validateOptions(schema, options, 'webpack Dev Server');
    this.compiler = compiler;
    this.options = options;
    // 1. 为一些选项提供默认参数
    normalizeOptions(this.compiler, this.options);
    // 2. 对 webpack compiler 进行一些修改  webpack-dev-server/lib/utils/updateCompiler.js
    //    - 如果设置了 hot 选项，自动给 webpack 配置 HotModuleReplacementPlugin
    //    - 注入一些客户端代码：webpack 的 websocket 客户端依赖 sockJS/websocket + websocket 客户端业务代码 + hot 模式下的 webpack/hot/dev-server
    updateCompiler(this.compiler, this.options);
    // 3. 添加一些 hooks 插件，这里主要关注 webpack compiler 的 done 钩子，即每次编译完成后的钩子 (编译完成触发 _sendStats 方法给客户端广播消息 )
    this.setupHooks();
    // 4. 实例化 express 服务器
    this.setupApp();
    // 5. 设置 webpack-dev-middleware，用于处理对静态资源的处理，后面解析
    this.setupDevMiddleware();
    // 6. 创建 HTTP 服务器
    this.createServer();
  }
 
 
  setupApp() {
    // Init express server
    // eslint-disable-next-line new-cap
    this.app = new express();
  }
 
  setupHooks() {
    const addHooks = (compiler) => {
      const { compile  } = compiler.hooks;
      done.tap('webpack-dev-server', (stats) => {
        this._sendStats(this.sockets, this.getStats(stats));
        this._stats = stats;
      });
    };
    addHooks(this.compiler);
  }
 
  setupDevMiddleware() {
    // middleware for serving webpack bundle
    this.middleware = webpackDevMiddleware(
      this.compiler,
      Object.assign({}, this.options, { logLevel: this.log.options.level })
    );
    this.app.use(this.middleware);
  }
 
 
  createServer() {
    this.listeningApp = http.createServer(this.app);
 
    this.listeningApp.on('error', (err) => {
      this.log.error(err);
    });
  }
 
 
  listen(port, hostname, fn) {
    this.hostname = hostname;
 
    return this.listeningApp.listen(port, hostname, (err) => {
      this.createSocketServer();
    });
  }
 
  createSocketServer() {
    const SocketServerImplementation = this.socketServerImplementation;
    this.socketServer = new SocketServerImplementation(this);
 
    this.socketServer.onConnection((connection, headers) => {
      // 连接后保存客户端连接
      this.sockets.push(connection);
 
      if (this.hot) {
        // hot 选项先广播一个 hot 类型的消息
        this.sockWrite([connection], 'hot');
      }
 
      this._sendStats([connection], this.getStats(this._stats), true);
    });
  }
 
 
  // eslint-disable-next-line
  sockWrite(sockets, type, data) {
    sockets.forEach((socket) => {
      this.socketServer.send(socket, JSON.stringify({ type, data }));
    });
  }
 
 
  // send stats to a socket or multiple sockets
  _sendStats(sockets, stats, force) {
    this.sockWrite(sockets, 'hash', stats.hash);
 
    if (stats.errors.length > 0) {
      this.sockWrite(sockets, 'errors', stats.errors);
    } else if (stats.warnings.length > 0) {
      this.sockWrite(sockets, 'warnings', stats.warnings);
    } else {
      this.sockWrite(sockets, 'ok');
    }
  }
}
```

这部分代码稍长，主逻辑都在构造函数里。

在构造函数中进行参数校验，参数缺省值处理，注入客户端代码，绑定 webpack compiler 钩子，这里主要关注是 done 钩子，(在 webpack compiler 实例每次触发编译完成后就会进行 webscoket 广播 webpack 的编译信息)。实例化 express 服务器，添加 webpack-dev-middleware 中间件用于处理静态资源的请求，然后初始化 HTTP 服务器。

我们在上面的` webpack-dev-server.js` 中调用的 `listen` 方法就是开始监听配置的端口，监听回调里再初始化 websocket 的服务端。代码执行到这已经完成了服务器端所有的逻辑，但是 webpack 还没有启动编译，用户打开浏览器后请求设置的IP和端口服务端又是怎么处理的呢？这部分暂时被我们略过了，这部分就是 `webpack-dev-middleware` 处理的内容了。

## webapck-dev-middleware 初始化
```js
// webpack-dev-middleware/index.js
module.exports = function wdm(compiler, opts) {
  const options = Object.assign({}, defaults, opts);
  // 1. 初始化 context
  const context = createContext(compiler, options);
 
  // start watching
  if (!options.lazy) {
    // 2. 启动 webpack 编译
    context.watching = compiler.watch(options.watchOptions, (err) => {
      if (err) {
        context.log.error(err.stack || err);
        if (err.details) {
          context.log.error(err.details);
        }
      }
    });
  } else {
    // lazy 模式是请求过来一次才webpack编译一次, 这里不关注
  }
 
  // 3. 替换 webpack 默认的 outputFileSystem 为 memory-fs, 存取都在内存上操作
  // fileSystem = new MemoryFileSystem();
  // compiler.outputFileSystem = fileSystem;
  setFs(context, compiler);
 
  // 3. 执行 middleware 函数返回真正的 middleware
  return middleware(context);
};
```
wdm 函数返回结果是 express 标准的 middleware 用于处理浏览器静态资源的请求。执行过程中显示初始化了一个 context 对象，默认非 lazy 模式，开启了 webpack 的 watch 模式开始启动编译。

然后将 compiler 的原来基于 fs 模块的 outputFileSystem 替换成 memory-fs模块的实例。memory-fs 是实现了 node fs api 的**基于内存的 fileSystem**，这意味着 webpack **编译后的资源不会被输出到硬盘而是内存**。最后将真正处理请求的 middleware 返回装载在 express 上。

## webapck-dev-middleware 处理请求
当用户在浏览器打开配置的IP和端口，如 https://localhost:8080 ，请求就会被 middleware 处理。middleware 使用 memory-fs 从内存中读到请求的资源返回给客户端。
```js
// webpack-dev-middleware/lib/middleware.js
module.exports = function wrapper(context) {
  return function middleware(req, res, next) {
    // 1. 根据请求的 URL 地址，得到绝对路径的 webpack 输出的资源路径地址
    let filename = getFilenameFromUrl(
      context.options.publicPath,
      context.compiler,
      req.url
    );
 
    return new Promise((resolve) => {
      handleRequest(context, filename, processRequest, req);
      // eslint-disable-next-line consistent-return
      function processRequest() {
 
        // 2.从内存读取到资源内容
        let content = context.fs.readFileSync(filename);
 
        // 3. 返回给客户端
        if (res.send) {
          res.send(content);
        } else {
          res.end(content);
        }
        resolve();
      }
    });
  };
};
```

## webscoket 通信
当我们编辑了源代码，触发 webpack 重新编译，编译完成后执行 done 钩子上的回调。具体可参考上面 `Server.js` 中` setupHooks `方法。`_sendStats `方法会先广播一个类型为` hash` 的消息，然后再根据编译信息广播` warnings/errors/ok` 消息。这里我们只关注正常流程 `ok `消息。

我们已经很熟悉客户端接收到更新后都会对应用进行 Reload 来获取更好的开发体验。具体是 liveReload（刷新整个页面）还是 hotReload（更新改动过的模块）就取决于我们传入的 hot 选项。

以下代码就是我们在上面就讲到的在 webpack 编译的时候注入到 bundle.js 进去的。当用户打开页面预览时，这些代码就会自动执行。

```js
// webpack-dev-server/client/index.js
var onSocketMessage = {
  hot: function hot() {
    options.hot = true;
    log.info('[WDS] Hot Module Replacement enabled.');
  },
  liveReload: function liveReload() {
    options.liveReload = true;
    log.info('[WDS] Live Reloading enabled.');
  },
  hash: function hash(_hash) {
    status.currentHash = _hash;
  },
  ok: function ok() {
    if (options.initial) {
      return options.initial = false;
    } // eslint-disable-line no-return-assign
 
 
    reloadApp(options, status);
  }
};
socket(socketUrl, onSocketMessage);
```

`client/index.js` 主要就是初始化了 webscoket 客户端，然后为不同的消息类型设置了相应的回调函数。

在前面` Server.js `中我们看到如果 `hot` 选项为 true 时，当 websocket 客户端连接到服务端，服务端会先广播一个 `hot `类型的消息，客户端接收到后会把 `options` 对象的` hot` 设置为 true。

服务端在每次编译后都会广播 `hash `消息，客户端接收到后就会将这个webpack 编译产生的 `hash `值暂存起来。编译成功如果没有 warning 也没有 error 就会广播 `ok `消息，客户端接收到` ok` 消息就会执行` ok `回调函数中的 `reloadApp` 刷新应用。

## websocket消息处理
```js
// webpack-dev-server/client/utils/reloadApp.js
 
function reloadApp(_ref, _ref2) {
  var hotReload = _ref.hotReload,
      hot = _ref.hot,
      liveReload = _ref.liveReload,
      currentHash = _ref2.currentHash;
 
  if (hot) {
    log.info('[WDS] App hot update...');
 
    var hotEmitter = require('webpack/hot/emitter');
 
    hotEmitter.emit('webpackHotUpdate', currentHash);
  }
  else if (liveReload) {
      var rootWindow = self; // use parent window for reload (in case we're in an iframe with no valid src)
 
      var intervalId = self.setInterval(function () {
        if (rootWindow.location.protocol !== 'about:') {
          // reload immediately if protocol is valid
          applyReload(rootWindow, intervalId);
        } else {
          rootWindow = rootWindow.parent;
 
          if (rootWindow.parent === rootWindow) {
            // if parent equals current window we've reached the root which would continue forever, so trigger a reload anyways
            applyReload(rootWindow, intervalId);
          }
        }
      });
    }
 
  function applyReload(rootWindow, intervalId) {
    clearInterval(intervalId);
    log.info('[WDS] App updated. Reloading...');
    // 调用reload
    rootWindow.location.reload();
  }
}
```

## Hot Module Replacement
### 触发 hot check
如果设置了` hot: true `客户端就会引入` webpack/hot/emitter`，触发一个 `webpackHotUpdate `事件，将` hash `值传递过去。这个` webpack/hot/emitter` 我们查阅 webpack 源码看到其实就是 node 的` events` 模块。我们暂时不关注这个事件会触发什么回调后面再具体再看。如果没有设置 `hot: true`。那么就是使用 `liveReload `模式，`liveReload` 就比较无脑，**直接刷新整个页面**。

再回到上一个问题，到底是在哪里接收` webpackHotUpdate `事件并处理的呢？就是` webpack/hot/dev-server.js` 中处理的。在这里会去**检查是否可以更新**，如果更新失败就会++刷新整个页面++来**降级**实现代码更新的功能。其实我们回过头来看看这样降级也是必须的，如果更新失败，源码更新了，而客户端的代码却没更新，这样显然是不合理的。

```js
 var lastHash;
 var upToDate = function upToDate() {
  return lastHash.indexOf(__webpack_hash__) >= 0;
 };
 var log = require("./log");
    // 2. 检查更新
 var check = function check() {
    // 3. 具体的检查逻辑
  module.hot
   .check(true)
   .then(function(updatedModules) {
        // 3.1 更新成功
   })
   .catch(function(err) {
    var status = module.hot.status();
        // 3.2 更新失败，降级为重新刷新整个应用
    if (["abort", "fail"].indexOf(status) >= 0) {
     log(
      "warning",
      "[HMR] Cannot apply update. Need to do a full reload!"
     );
     window.location.reload();
    } else {
     log("warning", "[HMR] Update failed: " + log.formatError(err));
    }
   });
 };
 var hotEmitter = require("./emitter");
  // 1. 注册事件回调
 hotEmitter.on("webpackHotUpdate", function(currentHash) {
  lastHash = currentHash;
  if (!upToDate() && module.hot.status() === "idle") {
   log("info", "[HMR] Checking for updates on the server...");
   check();
  }
 });
 
```

### 模块更新依赖判断
`module.hot.check `方法位于` webpack/lib/HotModuleReplacement.runtime.js `中，是 webpack 内置的 `HotModuleReplacementPlugin `注入在 webpack bootstrap runtime 中的。

所以 check 方法主要做了什么呢，这里提前总结一下。在 webpack 使用了 `HotModuleReplacementPlugin` 编译时，每次增量编译就会**多产出两个文件**，形如`c390bbe0037a0dd079a6.hot-update.json`，`main.c390bbe0037a0dd079a6.hot-update.js`，分别是:
1. 描述` chunk` 更新的` manifest`文件
1. 更新过后的` chunk `文件

那么浏览器端调用` hotDownloadManifest `方法去下载模块更新的 `manifest.json` 文件，然后调用` hotDownloadUpdateChunk` 方法使用 jsonp 的方式下载需要更新的 `chunk`。

`hotDownloadUpdateChunk `下载完成后调用` webpackHotUpdate` 回调。回调内拿到更新的模块，然后从模块自身开始进行冒泡，如果发现**只要有一条**祖先路径没有 accept 这次改动就直接刷新页面实行降级强制更新, 如果有被 accept, 就会替换掉原来 webpack runtime 里 module 里旧的模块，然后再执行 accept 的 callback 进行更新。为什么要执行这样的判断呢？


```js
function hotCheck(apply) {
  // 1. 拿这次编译后的 hash 请求服务器，拿到结构为 {c: {main: true} h: "ac69ee760bb48d5db5f5"} 的数据
  return hotDownloadManifest(hotRequestTimeout).then(function(update) {
    hotAvailableFilesMap = update.c;
    hotUpdateNewHash = update.h;
 
    // 2. 生成一个 defered promise，供上面提到的 promise 链消费
    var promise = new Promise(function(resolve, reject) {
      hotDeferred = {
        resolve: resolve,
        reject: reject
      };
    });
 
    hotUpdate = {};
    // 3. 这个方法里面调用的就是 hotDownloadUpdateChunk，就是发起一个 jsonp 请求更新过后的 chunk，jsonp的回调是 HMR runtime 里的 webpackHotUpdate
    {
      hotEnsureUpdateChunk(chunkId);
    }
 
    return promise;
  });
}
```
`hotCheck `方法就是和服务器进行通信拿到更新过后的` chunk`，下载好` chunk` 后就开始执行 HMR runtime 里的` webpackHotUpdate `回调。

```js
window["webpackHotUpdate"] = function webpackHotUpdateCallback(chunkId, moreModules) {
 hotAddUpdateChunk(chunkId, moreModules);
 if (parentHotUpdateCallback) parentHotUpdateCallback(chunkId, moreModules);
} ;
```

经过一系列方法调用然后来到` hotApplyInternal` 方法，这个方法把更新过后的模块 apply 到业务中，整个方法比较长，就不完整贴出来了。这里拿出核心的部分，
```js
for (var id in hotUpdate) {
    if (Object.prototype.hasOwnProperty.call(hotUpdate, id)) {
        var result;
        if (hotUpdate[id]) {
            result = getAffectedStuff(moduleId);
        } else {
            result = {
                type: "disposed",
                moduleId: id
            };
        }
        switch (result.type) {
            case "self-declined":
            case "declined":
            case "unaccepted":
                if (options.onUnaccepted) options.onUnaccepted(result);
                if (!options.ignoreUnaccepted)
                    abortError = new Error(
                        "Aborted because " + moduleId + " is not accepted" + chainInfo
                    );
                break;
            case "accepted":
                if (options.onAccepted) options.onAccepted(result);
                doApply = true;
                break;
            case "disposed":
                break;
            default:
                throw new Error("Unexception type " + result.type);
        }
    }
}
```
把更新过的模块进行遍历，找到被该模块影响到的**祖先模块**，返回一个结果，如果结果标识为 unaccepted 就会被抛出错误，然后走到 `webpack/hot/dev-server.js` 里的 catch 进行页面级刷新。如果被 accept 的话就会执行后面的 apply 的逻辑。

```js
function getAffectedStuff(updateModuleId) {
  var outdatedModules = [updateModuleId];
  var outdatedDependencies = {};
 
  var queue = outdatedModules.map(function(id) {
    return {
      chain: [id],
      id: id
    };
  });
  // 1. 遍历 queue
  while (queue.length > 0) {
    var queueItem = queue.pop();
    var moduleId = queueItem.id;
    var chain = queueItem.chain;
    // 2. 找到改模块的旧版本
    module = installedModules[moduleId];
 
    // 3. 如果到根模块了，返回 unaccepted
    if (module.hot._main) {
      return {
        type: "unaccepted",
        chain: chain,
        moduleId: moduleId
      };
    }
    // 4. 遍历父模块
    for (var i = 0; i < module.parents.length; i++) {
      var parentId = module.parents[i];
      var parent = installedModules[parentId];
 
      // 5. 如果父模块处理了模块变更的话就跳过，继续检查
      if (parent.hot._acceptedDependencies[moduleId]) {
        continue;
      }
      outdatedModules.push(parentId);
      // 6. 没跳过的话推入队列，继续检查
      queue.push({
        chain: chain.concat([parentId]),
        id: parentId
      });
    }
  }
 
  // 7.如果所有依赖路径都有被 accept 就返回 accepted
  return {
    type: "accepted",
    moduleId: updateModuleId,
    outdatedModules: outdatedModules,
    outdatedDependencies: outdatedDependencies
  };
}
```
### module apply
看过 webpack runtime 代码之后我们知道 runtime 里声明了 `installedModules` 这个变量，里面缓存了所有被` __webpack_require__` 调用后加载过的模块，还有 `modules `这个变量存储了所有模块。（如果不了解 webpack runtime 可以先了解 webpack runtime 的执行机制）。如果模块有被 accept 的话，那么就会从 `installedModules` 里**删掉旧的模块**，**把模块从父子依赖中删除**，然后把 modules 里面的模块替换成新的模块。

hotApply
```js
// remove module from cache
delete installedModules[moduleId];
 
 
// insert new code
for (moduleId in appliedUpdate) {
    if (Object.prototype.hasOwnProperty.call(appliedUpdate, moduleId)) {
        modules[moduleId] = appliedUpdate[moduleId];
    }
}
```

```js
// webpack/lib/HotModuleReplacement.runtime
function hotApply() {
    // ...
    var idx;
    var queue = outdatedModules.slice();
    while(queue.length > 0) {
        moduleId = queue.pop();
        module = installedModules[moduleId];
        // ...
        // remove module from cache
        delete installedModules[moduleId];
        // when disposing there is no need to call dispose handler
        delete outdatedDependencies[moduleId];
        // remove "parents" references from all children
        for(j = 0; j < module.children.length; j++) {
            var child = installedModules[module.children[j]];
            if(!child) continue;
            idx = child.parents.indexOf(moduleId);
            if(idx >= 0) {
                child.parents.splice(idx, 1);
            }
        }
    }
    // ...
    // insert new code
    for(moduleId in appliedUpdate) {
        if(Object.prototype.hasOwnProperty.call(appliedUpdate, moduleId)) {
            modules[moduleId] = appliedUpdate[moduleId];
        }
    }
    // ...
}
```
从上面 hotApply 方法可以看出，模块热替换主要分三个阶段，第一个阶段是找出 `outdatedModules` 和 `outdatedDependencies`.

第二个阶段从缓存中删除过期的模块和依赖，如下：
```js
delete installedModules[moduleId];
delete outdatedDependencies[moduleId];
```

第三个阶段是将新的模块添加到 `modules` 中，当下次调用 `__webpack_require__` (webpack 重写的 require 方法)方法的时候，就是获取到了新的模块代码了。




这样仅仅完成了模块的替换，还没有执行过新模块代码，也就是没被 `__webpack_require__` 调用过。对于 ES Module，新模块代码的执行是在 accept 函数的 callback 里被 webpack 自动插入代码执行的。使用 `require()` 引入的模块不会被自动执行。


### 业务代码需要做什么
当用新的模块代码替换老的模块后，但是我们的业务代码并不能知道代码已经发生变化，也就是说，当 hello.js 文件修改后，我们需要在 index.js 文件中调用 HMR 的 accept 方法，添加模块更新后的处理函数，及时将 hello 方法的返回值插入到页面中。代码如下：
```js
// index.js
if(module.hot) {
    module.hot.accept('./hello.js', function() {
        div.innerHTML = hello()
    })
}
```

webpack会将这个代码改造
```js
if(module.hot) {
    module.hot.accept('./App', function() {
        console.log('accepted')
    })
}
```
会被webpack改造为
```js
if(true) {
    module.hot.accept("./src/App.js", function(__WEBPACK_OUTDATED_DEPENDENCIES__) {
      _App__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__("./src/App.js");
      (function() {
        console.log('accepted')
      })(__WEBPACK_OUTDATED_DEPENDENCIES__);
    }.bind(this))
}
```

所以新模块的代码是在 accept 方法回调执行之前被执行的。引入了新代码后就可以执行我们的业务代码，这些业务代码一般都和框架相关，框架去处理模块的热更新逻辑。比如 `react-hot-loader, vue-loader`


## 总结
> 基本实现原理大致这样的，构建 bundle 的时候，加入一段 HMR runtime 的 js 和一段和服务沟通的 js 。文件修改会触发 webpack 重新构建，服务器通过websocket向浏览器发送更新消息，浏览器通过 jsonp 拉取更新的模块文件，jsonp 回调触发模块热替换逻辑。

最后总结一下，`webpack-dev-server `可以作为命令行工具使用，核心模块依赖是 `webpack `和 `webpack-dev-middleware`。`webapck-dev-server` 负责启动一个 express 服务器监听客户端请求；实例化` webpack compiler`；启动负责推送 webpack 编译信息的 webscoket 服务器；负责向 `bundle.js` 注入和服务端通信用的 webscoket 客户端代码和处理逻辑。`webapck-dev-middleware` 把` webpack compiler` 的 `outputFileSystem `改为 `in-memory fileSystem`；启动 `webpack watch` 编译；处理浏览器发出的静态资源的请求，把 webpack 输出到内存的文件响应给浏览器。

每次 webpack 编译完成后向客户端广播 ok 消息，客户端收到信息后根据是否开启 hot 模式使用 liveReload 页面级刷新模式或者 hotReload 模块热替换。hotReload 存在失败的情况，失败的情况下会降级使用页面级刷新。

开启 hot 模式，即启用 HMR 插件。hot 模式会向服务器请求更新过后的模块，然后对模块的父模块进行回溯，对依赖路径进行判断，如果每条依赖路径都配置了模块更新后所需的业务处理回调函数则是 accepted 状态，否则就降级刷新页面。判断 accepted 状态后对旧的缓存模块和父子依赖模块进行替换和删除，然后执行 accept 方法的回调函数，执行新模块代码，引入新模块，执行业务处理代码。


## 参考
[webpack-dev-server 运行原理分析](https://blog.csdn.net/xgangzai/article/details/113011078)

<!--https://github.com/careteenL/webpack-hmr-->
[搞懂webpack热更新原理](https://github.com/careteenL/webpack-hmr)