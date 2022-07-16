---
title: webpack核心原理
date: {{ date }}
tags: webpack
cover: https://ppoffice.github.io/hexo-theme-icarus/gallery/covers/vector_landscape_2.svg
excerpt: 非常细致的webpack原理，不过由于太过详细，有点难以记忆。
toc: true
---
# webpack核心原理

## 1. 构建的核心流程

Webpack 最核心的功能：

> At its core, **webpack** is a *static module bundler* for modern JavaScript applications.

也就是将各种类型的资源，包括图片、css、js等，转译、组合、拼接、生成 JS 格式的 bundler 文件。

![1d66a833-2841-4a8a-a91a-0da800fab306.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1d3e3165499409e925c2832e0566df4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

这个过程核心完成了 **内容转换 + 资源合并** 两种功能，实现上包含三个阶段：

**一、初始化阶段**：

1. **初始化参数**：从配置文件、 配置对象、Shell 参数中读取，与默认配置结合得出最终的参数
2. **创建编译器对象**：用上一步得到的参数创建 `Compiler` 对象
3. **初始化编译环境**：包括注入内置插件、注册各种模块工厂、初始化 RuleSet 集合、加载配置的插件等
4. **开始编译**：执行 `compiler` 对象的 `run` 方法
5. **确定入口**：根据配置中的 `entry` 找出所有的入口文件，调用 `compilition.addEntry` 将入口文件转换为 `dependence` 对象

**二、构建阶段**：

1. **编译模块(make)**：根据 `entry` 对应的 `dependence` 创建 `module` 对象，调用 `loader` 将模块转译为标准 JS 内容，调用 JS 解释器将内容转换为 AST 对象，从中找出该模块依赖的模块，再 递归 本步骤直到所有入口依赖的文件都经过了本步骤的处理
2. **完成模块编译**：上一步递归处理所有能触达到的模块后，得到了每个模块被翻译后的内容以及它们之间的 **依赖关系图**

**三、生成阶段**：

1. **输出资源(seal)**：根据入口和模块之间的依赖关系，组装成一个个包含多个模块的 `Chunk`，再把每个 `Chunk` 转换成一个单独的文件加入到输出列表，这步是可以修改输出内容的最后机会
2. **写入文件系统(emitAssets)**：在确定好输出内容后，根据配置确定输出的路径和文件名，把文件内容写入到文件系统

### 一些名词的解释

- `Entry`：编译入口，webpack 编译的起点
- `Compiler`：编译管理器，webpack 启动后会创建 `compiler` 对象，该对象一直存活知道结束退出
- `Compilation`：单次编辑过程的管理器，比如 `watch = true` 时，运行过程中只有一个 `compiler` 但每次文件变更触发重新编译时，都会创建一个新的 `compilation` 对象
- `Dependence`：依赖对象，webpack 基于该类型记录模块间依赖关系
- `Module`：webpack 内部所有资源都会以“module”对象形式存在，所有关于资源的操作、转译、合并都是以 “module” 为基本单位进行的
- `Chunk`：编译完成准备输出时，webpack 会将 `module` 按特定的规则组织成一个一个的 `chunk`，这些 `chunk` 某种程度上跟最终输出一一对应
- `Loader`：资源内容转换器，其实就是实现从内容 A 转换 B 的转换器
- `Plugin`：webpack构建过程中，会在特定的时机广播对应的事件，插件监听这些事件，在特定时间点介入编译过程。

webpack 编译过程都是围绕着这些关键对象展开的，更详细完整的信息，可以参考 [Webpack 知识图谱](https://juejin.cn/post/6948763207397965855) 。

### 一、初始化阶段

<img src="\images\d8ce6474c85247829475ba1f2ec9a047tplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp" alt="img"  />

- 将 `process.args + webpack.config.js` 合并成用户配置
- 调用 `validateSchema` 校验配置
- 调用 `getNormalizedWebpackOptions + applyWebpackOptionsBaseDefaults` 合并出最终配置
- 创建 `compiler` 对象
- 遍历用户定义的 `plugins` 集合，执行插件的 `apply` 方法
- 调用 `new WebpackOptionsApply().process` 方法，加载各种内置插件

主要逻辑集中在 `WebpackOptionsApply` 类，webpack 内置了数百个插件，这些插件并不需要我们手动配置，`WebpackOptionsApply` 会在初始化阶段根据配置内容动态注入对应的插件，包括：

- 注入 `EntryOptionPlugin` 插件，处理 `entry` 配置
- 根据 `devtool` 值判断后续用那个插件处理 `sourcemap`，可选值：`EvalSourceMapDevToolPlugin`、`SourceMapDevToolPlugin`、`EvalDevToolModulePlugin`
- 注入 `RuntimePlugin` ，用于根据代码内容动态注入 webpack 运行时

到这里，`compiler` 实例就被创建出来了，相应的环境参数也预设好了，紧接着开始调用 `compiler.compile` 函数：

```js
// 取自 webpack/lib/compiler.js 
compile(callback) {
    const params = this.newCompilationParams();
    this.hooks.beforeCompile.callAsync(params, err => {
      // ...
      const compilation = this.newCompilation(params);
      this.hooks.make.callAsync(compilation, err => {
        // ...
        this.hooks.finishMake.callAsync(compilation, err => {
          // ...
          process.nextTick(() => {
            compilation.finish(err => {
              compilation.seal(err => {...});
            });
          });
        });
      });
    });
  }
```

Webpack 架构很灵活，但代价是牺牲了源码的直观性，比如说上面说的初始化流程，从创建 `compiler` 实例到调用 `make` 钩子，逻辑链路很长：

- 启动 webpack ，触发 `lib/webpack.js` 文件中 `createCompiler` 方法
- `createCompiler` 方法内部调用 `WebpackOptionsApply` 插件
- `WebpackOptionsApply` 定义在 `lib/WebpackOptionsApply.js` 文件，内部根据 `entry` 配置决定注入 `entry` 相关的插件，包括：`DllEntryPlugin`、`DynamicEntryPlugin`、`EntryPlugin`、`PrefetchPlugin`、`ProgressPlugin`、`ContainerPlugin`
- `Entry` 相关插件，如 `lib/EntryPlugin.js` 的 `EntryPlugin` 监听 `compiler.make` 钩子
- `lib/compiler.js` 的 `compile` 函数内调用 `this.hooks.make.callAsync`
- 触发 `EntryPlugin` 的 `make` 回调，在回调中执行 `compilation.addEntry` 函数
- `compilation.addEntry` 函数内部经过一坨与主流程无关的 `hook` 之后，再调用 `handleModuleCreate` 函数，正式开始构建内容

### 二、构建阶段

你有没有思考过这样的问题：

- Webpack 编译过程会将源码解析为 AST 吗？webpack 与 babel 分别实现了什么？
- Webpack 编译过程中，如何识别资源对其他资源的依赖？
- 相对于 grunt、gulp 等流式构建工具，为什么 webpack 会被认为是新一代的构建工具？


这些问题，基本上在构建阶段都能看出一些端倪。构建阶段从 `entry` 开始**递归解析**资源与资源的依赖，在 `compilation` 对象内逐步构建出 `module` 集合以及 `module` 之间的依赖关系，核心流程：

![img](\images\69c49d365f894fc19e42bce73d674b08tplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)

解释一下，构建阶段从入口文件开始：(**递归过程**)

1. 调用 `handleModuleCreate` ，根据文件类型构建 `module` 子类
2. 调用 [loader-runner](https://link.juejin.cn?target=https%3A%2F%2Fwww.npmjs.com%2Fpackage%2Floader-runner) 仓库的 `runLoaders` 转译 `module` 内容，通常是从各类资源类型**转译为 JavaScript 文本**
3. 调用 [acorn](https://link.juejin.cn?target=https%3A%2F%2Fwww.npmjs.com%2Fpackage%2Facorn) 将 **JS 文本解析为AST**
4. 遍历 AST，**触发各种钩子**
5. 1. 在 `HarmonyExportDependencyParserPlugin` 插件监听 `exportImportSpecifier` 钩子，**解读 JS 文本对应的资源依赖**
   2. 调用 `module` 对象的 `addDependency` 将依赖对象加入到 `module` 依赖列表中
6. AST 遍历完毕后，调用 `module.handleParseResult` **处理模块依赖**
7. 对于 `module` 新增的依赖，调用 `handleModuleCreate` ，控制流回到第一步
8. 所有依赖都解析完毕后，构建阶段结束

这个过程中**数据流** `module => ast => dependences => module` ，先转 AST 再从 AST 找依赖。这就要求 `loaders` 处理完的最后结果必须是可以被 acorn 处理的标准 JavaScript 语法，比如说对于图片，需要从图像二进制转换成类似于 `export default "data:image/png;base64,xxx"` 这类 **base64 格式**或者 `export default "http://xxx"` 这类 **url 格式**。

`compilation` 按这个流程递归处理，逐步解析出每个模块的内容以及 `module` 依赖关系，后续就可以根据这些内容打包输出。

#### 示例：层级递进

假如有如下图所示的文件依赖树：

![img](\images\16ff09e71aec484eb394696ccb9d60a0tplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)

其中 `index.js` 为 `entry` 文件，依赖于 a/b 文件；a 依赖于 c/d 文件。初始化编译环境之后，`EntryPlugin` 根据 `entry` 配置找到 `index.js` 文件，调用 `compilation.addEntry` 函数触发构建流程，构建完毕后内部会生成这样的数据结构：

![img](\images\d9c9ca778cf545de9099c463723d8f1btplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)此时得到 `module[index.js]` 的内容以及对应的依赖对象 `dependence[a.js]` 、`dependence[b.js]` 。OK，这就得到下一步的线索：a.js、b.js，根据上面流程图的逻辑继续调用 `module[index.js]` 的 `handleParseResult` 函数，继续处理 a.js、b.js 文件，递归上述流程，进一步得到 a、b 模块：

![img](\images\0a43a123b172418aba4019dbd79e064dtplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)

从 a.js 模块中又解析到 c.js/d.js 依赖，于是再再继续调用 `module[a.js]` 的 `handleParseResult` ，再再递归上述流程：

![img](\images\0c22cb909cbd46cf98bbc64fdec5065ftplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)

到这里解析完所有模块后，发现没有更多新的依赖，就可以继续推进，进入下一步。

#### 总结

回顾章节开始时提到的问题：

- Webpack 编译过程会将源码解析为 AST 吗？webpack 与 babel 分别实现了什么？
  - 构建阶段会读取源码，解析为 AST 集合。
  - Webpack 读出 AST 之后仅遍历 AST 集合；babel 则对源码做等价转换
- Webpack 编译过程中，如何识别资源对其他资源的依赖？
  - Webpack 遍历 AST 集合过程中，识别 `require/ import` 之类的导入语句，确定模块对其他资源的依赖关系
- 相对于 grant、gulp 等流式构建工具，为什么 webpack 会被认为是新一代的构建工具？
  - Grant、Gulp 仅执行开发者预定义的任务流；而 webpack 则深入处理资源的内容，功能上更强大



### 三、生成阶段

#### 基本流程

<u>构建阶段围绕 `module` 展开，生成阶段则围绕 `chunks` 展开</u>。

经过构建阶段之后，webpack 得到足够的模块内容与模块关系信息，接下来开始生成最终资源了。代码层面，就是开始执行 `compilation.seal` 函数：

```js
// 取自 webpack/lib/compiler.js 
compile(callback) {
    const params = this.newCompilationParams();
    this.hooks.beforeCompile.callAsync(params, err => {
      // ...
      const compilation = this.newCompilation(params);
      this.hooks.make.callAsync(compilation, err => {
        // ...
        this.hooks.finishMake.callAsync(compilation, err => {
          // ...
          process.nextTick(() => {
            compilation.finish(err => {
              **compilation.seal**(err => {...});
            });
          });
        });
      });
    });
  }
```

`seal` 原意密封、上锁，我个人理解在 webpack 语境下接近于 **“将模块装进蜜罐”** 。`seal` 函数主要完成从 `module` 到 `chunks` 的转化，核心流程：

![img](\images\ae16969ecfe44bb8aef5f628fdd60f9btplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)

简单梳理一下：

1. 构建本次编译的 `ChunkGraph` 对象；
2. 遍历 `compilation.modules` 集合，将 `module` 按 `entry/动态引入` 的规则分配给不同的 `Chunk` 对象；
3. `compilation.modules` 集合遍历完毕后，得到完整的 `chunks` 集合对象，调用 `createXxxAssets` 方法
4. `createXxxAssets` 遍历 `module/chunk` ，调用 `compilation.emitAssets` 方法将资 `assets` 信息记录到 `compilation.assets` 对象中
5. 触发 `seal` 回调，控制流回到 `compiler` 对象


这一步的关键逻辑是将 `module` 按规则组织成 `chunks` ，webpack 内置的 `chunk` 封装规则比较简单：

- `entry` 及 entry 触达到的模块，组合成一个 `chunk`
- 使用动态引入语句引入的模块，各自组合成一个 `chunk`

`chunk` 是输出的基本单位，默认情况下这些 `chunks` 与最终输出的资源一一对应，那按上面的规则大致上可以推导出一个 `entry` 会对应打包出一个资源，而通过动态引入语句引入的模块，也对应会打包出相应的资源，我们来看个示例。

#### 示例：多入口打包

假如有这样的配置：

```js
const path = require("path");

module.exports = {
  mode: "development",
  context: path.join(__dirname),
  entry: {
    a: "./src/index-a.js",
    b: "./src/index-b.js",
  },
  output: {
    filename: "[name].js",
    path: path.join(__dirname, "./dist"),
  },
  devtool: false,
  target: "web",
  plugins: [],
};
```

实例配置中有两个入口，对应的文件结构：

![img](\images\857866c0c60f44cfb8a87e67723c017btplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)

`index-a` 依赖于c，且动态引入了 e；`index-b` 依赖于 c/d 。根据上面说的规则：

- **`entry` 及entry触达到的模块，组合成一个 chunk**
- **使用动态引入语句引入的模块，各自组合成一个 chunk**

生成的 `chunks` 结构为：

![img](\images\9b4d576b67404c28b01b6390f60d4e3ftplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)

也就是根据依赖关系，`chunk[a]` 包含了 `index-a/c` 两个模块；`chunk[b]` 包含了 `c/index-b/d` 三个模块；`chunk[e-hash]` 为动态引入 `e` 对应的 chunk。

不知道大家注意到没有，`chunk[a]` 与 `chunk[b]` 同时包含了 c，这个问题放到具体业务场景可能就是，一个多页面应用，所有页面都依赖于相同的基础库，那么这些所有页面对应的 `entry` 都会包含有基础库代码，这岂不浪费？为了解决这个问题，webpack 提供了一些插件如 `CommonsChunkPlugin` 、`SplitChunksPlugin`，在基本规则之外进一步优化 `chunks` 结构。

#### `SplitChunksPlugin` 的作用

`SplitChunksPlugin` 是 webpack 架构高扩展的一个绝好的示例，我们上面说了 webpack 主流程里面是按 `entry / 动态引入` 两种情况组织 `chunks` 的，这必然会引发一些不必要的重复打包，webpack 通过插件的形式解决这个问题。

回顾 `compilation.seal` 函数的代码，大致上可以梳理成这么4个步骤：

1. 遍历 `compilation.modules` ，记录下模块与 `chunk` 关系
2. 触发各种模块优化钩子，这一步优化的主要是**模块依赖关系**
3. 遍历 `module` 构建 chunk 集合
4. 触发各种优化钩子

![image (3).png](\images\57de01d125a349f1ba58e56024063f85tplv-k3u1fbpfcp-zoom-in-crop-mark1304000-165461470902315.awebp)



上面 1-3 都是预处理 + chunks 默认规则的实现，不在我们讨论范围，这里重点关注第4个步骤触发的 `optimizeChunks` 钩子，这个时候已经跑完主流程的逻辑，得到 `chunks` 集合，`SplitChunksPlugin` 正是使用这个钩子，分析 `chunks` 集合的内容，按配置规则增加一些通用的 chunk ：


```js
module.exports = class SplitChunksPlugin {
  constructor(options = {}) {
    // ...
  }

  _getCacheGroup(cacheGroupSource) {
    // ...
  }

  apply(compiler) {
    // ...
    compiler.hooks.thisCompilation.tap("SplitChunksPlugin", (compilation) => {
      // ...
      compilation.hooks.optimizeChunks.tap(
        {
          name: "SplitChunksPlugin",
          stage: STAGE_ADVANCED,
        },
        (chunks) => {
          // ...
        }
      );
    });
  }
};
```

理解了吗？webpack 插件架构的高扩展性，使得整个编译的主流程是可以固化下来的，分支逻辑和细节需求“外包”出去由第三方实现，这套规则架设起了庞大的 webpack 生态，关于插件架构的更多细节，下面 `plugin` 部分有详细介绍，这里先跳过。

#### 写入文件系统

经过构建阶段后，`compilation` 会获知资源模块的内容与依赖关系，也就知道“输入”是什么；而经过 `seal` 阶段处理后， `compilation` 则获知资源输出的图谱，也就是知道怎么“输出”：哪些模块跟那些模块“绑定”在一起输出到哪里。`seal` 后大致的数据结构：

```js
compilation = {
  // ...
  modules: [
    /* ... */
  ],
  chunks: [
    {
      id: "entry name",
      files: ["output file name"],
      hash: "xxx",
      runtime: "xxx",
      entryPoint: {xxx}
      // ...
    },
    // ...
  ],
};
```

`seal` 结束之后，紧接着调用 `compiler.emitAssets` 函数，函数内部调用 `compiler.outputFileSystem.writeFile` 方法将 `assets` 集合写入文件系统，实现逻辑比较曲折，但是与主流程没有太多关系，所以这里就不展开讲了。

#### 资源形态流转

上面已经把逻辑层面的构造主流程梳理完了，这里结合**资源形态流转**的角度重新考察整个过程，加深理解：

![img](\images\fe923c3d560241d38815d549786eab4ftplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)

- `compiler.make` 阶段：

  - `entry` 文件以 `dependence` 对象形式加入 `compilation` 的依赖列表，`dependence` 对象记录有 `entry` 的类型、路径等信息

  - 根据 `dependence` 调用对应的工厂函数创建 `module` 对象，之后读入 `module` 对应的文件内容，调用 `loader-runner` 对内容做转化，转化结果若有其它依赖则继续读入依赖资源，重复此过程直到所有依赖均被转化为 `module`

- `compilation.seal` 阶段：

  - 遍历 `module` 集合，根据 `entry` 配置及引入资源的方式，将 `module` 分配到不同的 `chunk`

  - 遍历 `chunk` 集合，调用 `compilation.emitAsset` 方法标记 `chunk` 的输出规则，即转化为 `assets` 集合

- `compiler.emitAssets` 阶段：
  - 将 `assets` 写入文件系统



## 2. loader的作用

Loader 的作用和实现比较简单，容易理解。回顾 loader 在编译流程中的生效的位置：

![img](\images\1c11e6fc73724a2f9197676c538c8173tplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)

流程图中， `runLoaders` 会调用用户所配置的 loader 集合读取、转译资源，此前的内容可以千奇百怪，但转译之后理论上应该输出标准 JavaScript 文本或者 AST 对象，webpack 才能继续处理模块依赖。

理解了这个基本逻辑之后，loader 的职责就比较清晰了，不外乎是将内容 A 转化为内容 B，但是在具体用法层面还挺多讲究的，有 pitch、pre、post、inline 等概念用于应对各种场景。

为了帮助理解，这里补充一个示例： [Webpack 案例 -- vue-loader 原理分析](https://juejin.cn/post/6937125495439900685)。

细节参考文章：https://juejin.cn/post/6992754161221632030

## 3. plugin架构

### What: 什么是插件

从形态上看，插件通常是一个带有 `apply` 函数的类：

```js
class SomePlugin {
    apply(compiler) {
    }
}
```

`apply` 函数运行时会得到参数 `compiler` ，以此为起点可以调用 `hook` 对象注册各种钩子回调，例如： `compiler.hooks.make.tapAsync` ，这里面 `make` 是钩子名称，`tapAsync` 定义了钩子的调用方式，webpack 的插件架构基于这种模式构建而成，插件开发者可以使用这种模式在钩子回调中，插入特定代码。webpack 各种内置对象都带有 `hooks` 属性，比如 `compilation` 对象：


```js
class SomePlugin {
    apply(compiler) {
        compiler.hooks.thisCompilation.tap('SomePlugin', (compilation) => {
            compilation.hooks.optimizeChunkAssets.tapAsync('SomePlugin', ()=>{});
        })
    }
}
```

钩子的核心逻辑定义在 [Tapable](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fwebpack%2Ftapable) 仓库，内部定义了如下类型的钩子：

```js
const {
        SyncHook,
        SyncBailHook,
        SyncWaterfallHook,
        SyncLoopHook,
        AsyncParallelHook,
        AsyncParallelBailHook,
        AsyncSeriesHook,
        AsyncSeriesBailHook,
        AsyncSeriesWaterfallHook
 } = require("tapable");
```

不同类型的钩子根据其并行度、熔断方式、同步异步，调用方式会略有不同，插件开发者需要根据这些的特性，编写不同的交互逻辑



### When: 什么时候会触发钩子

了解 webpack 插件的基本形态之后，接下来需要弄清楚一个问题：webpack 会在什么时间节点触发什么钩子？

先看几个例子：

- `compiler.hooks.compilation` ：

  - 时机：启动编译创建出 compilation 对象后触发

  - 参数：当前编译的 compilation 对象
  - 示例：很多插件基于此事件获取 compilation 实例

- `compiler.hooks.make`：

  - 时机：正式开始编译时触发
  - 参数：同样是当前编译的 `compilation` 对象
  - 示例：webpack 内置的 `EntryPlugin` 基于此钩子实现 `entry` 模块的初始化

- `compilation.hooks.optimizeChunks` ：

  - 时机： `seal` 函数中，`chunk` 集合构建完毕后触发
  - 参数：`chunks` 集合与 `chunkGroups` 集合
  - 示例： `SplitChunksPlugin` 插件基于此钩子实现 `chunk` 拆分优化

- `compiler.hooks.done`：

  - 时机：编译完成后触发
  - 参数： `stats` 对象，包含编译过程中的各类统计信息
  - 示例： `webpack-bundle-analyzer` 插件基于此钩子实现打包分析

不同钩子的触发时机、传递参数各不相同。

### How: 如何影响编译状态

解决上述两个问题之后，我们就能理解“如何将特定逻辑插入 webpack 编译过程”，接下来才是重点 —— 如何影响编译状态？强调一下，webpack 的插件体系与平常所见的 订阅/发布 模式差别很大，是一种非常强耦合的设计，hooks 回调由 webpack 决定何时，以何种方式执行；而在 hooks 回调内部可以通过修改状态、调用上下文 api 等方式对 webpack 产生 **side effect**。

比如，`EntryPlugin` 插件：

```js
class EntryPlugin {
  apply(compiler) {
    compiler.hooks.compilation.tap(
      "EntryPlugin",
      (compilation, { normalModuleFactory }) => {
        compilation.dependencyFactories.set(
          EntryDependency,
          normalModuleFactory
        );
      }
    );

    compiler.hooks.make.tapAsync("EntryPlugin", (compilation, callback) => {
      const { entry, options, context } = this;

      const dep = EntryPlugin.createDependency(entry, options);
      compilation.addEntry(context, dep, options, (err) => {
        callback(err);
      });
    });
  }
}
```

上述代码片段调用了两个影响 `compilation` 对象状态的接口：

- `compilation.dependencyFactories.set`
- `compilation.addEntry`

操作的具体含义可以先忽略，这里要理解的重点是，webpack 会将上下文信息以参数或 `this` (compiler 对象) 形式传递给钩子回调，在回调中可以调用上下文对象的方法或者直接修改上下文对象属性的方式，对原定的流程产生 side effect。所以想纯熟地编写插件，除了要理解调用时机，还需要了解我们可以用哪一些api，例如：



## 4. Tapable

参考：https://juejin.cn/post/6844903713312604173

Webpack可以将其理解是一种基于事件流的编程范例，一个插件合集。

而将这些插件控制在webapck事件流上的运行的就是webpack自己写的基础类`Tapable`。

Tapable暴露出挂载`plugin`的方法，使我们能 将plugin控制在webapack事件流上运行（如下图）。后面我们将看到核心的对象 `Compiler`、`Compilation`等都是继承于`Tabable`类。(如下图所示)

![img](\images\2a78e89c5f5440f09dc13ab83e82d0bbtplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)

### Tapable是什么？

[tapable库](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fwebpack%2Ftapable)暴露了很多Hook（钩子）类，为插件提供挂载的钩子。

```js
const {
	SyncHook,
	SyncBailHook,
	SyncWaterfallHook,
	SyncLoopHook,
	AsyncParallelHook,
	AsyncParallelBailHook,
	AsyncSeriesHook,
	AsyncSeriesBailHook,
	AsyncSeriesWaterfallHook
 } = require("tapable");

```

![img](\images\52687588cbc746aa9da3d9349aea1dcetplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)

### Tabable 用法

参考：https://juejin.cn/post/6844903713312604173

**1.new Hook 新建钩子**

- tapable 暴露出来的都是类方法，new 一个类方法获得我们需要的钩子。
- class 接受数组参数options，非必传。类方法会根据传参，接受同样数量的参数。

```js
const hook1 = new SyncHook(["arg1", "arg2", "arg3"]);
```

**2.使用 tap/tapAsync/tapPromise 绑定钩子**

tabpack提供了`同步`&`异步`绑定钩子的方法，并且他们都有`绑定事件`和`执行事件`对应的方法。

| Async                         | Sync       |
| ----------------------------- | ---------- |
| 绑定：tapAsync/tapPromise/tap | 绑定：tap  |
| 执行：callAsync/promise       | 执行：call |



**3.call/callAsync 执行绑定事件**

```js
const hook1 = new SyncHook(["arg1", "arg2", "arg3"]);

//绑定事件到webapck事件流
hook1.tap('hook1', (arg1, arg2, arg3) => console.log(arg1, arg2, arg3)) //1,2,3

//执行绑定的事件
hook1.call(1,2,3)
```

![img](\images\97f6960f5b304722904754d8317b12fbtplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)

#### 举个例子

- 定义一个Car方法，在内部hooks上新建钩子。分别是`同步钩子` accelerate、break（accelerate接受一个参数）、`异步钩子`calculateRoutes
- 使用钩子对应的`绑定和执行方法`
- calculateRoutes使用`tapPromise`可以返回一个`promise`对象。


```js
//引入tapable
const {
    SyncHook,
    AsyncParallelHook
} = require('tapable');

//创建类
class Car {
    constructor() {
        this.hooks = {
            accelerate: new SyncHook(["newSpeed"]),
            break: new SyncHook(),
            calculateRoutes: new AsyncParallelHook(["source", "target", "routesList"])
        };
    }
}

const myCar = new Car();

//绑定同步钩子
myCar.hooks.break.tap("WarningLampPlugin", () => console.log('WarningLampPlugin'));

//绑定同步钩子 并传参
myCar.hooks.accelerate.tap("LoggerPlugin", newSpeed => console.log(`Accelerating to ${newSpeed}`));

//绑定一个异步Promise钩子
myCar.hooks.calculateRoutes.tapPromise("calculateRoutes tapPromise", (source, target, routesList, callback) => {
    // return a promise
    return new Promise((resolve,reject)=>{
        setTimeout(()=>{
            console.log(`tapPromise to ${source}${target}${routesList}`)
            resolve();
        },1000)
    })
});

//执行同步钩子
myCar.hooks.break.call();
myCar.hooks.accelerate.call('hello');

console.time('cost');

//执行异步钩子
myCar.hooks.calculateRoutes.promise('i', 'love', 'tapable').then(() => {
    console.timeEnd('cost');
}, err => {
    console.error(err);
    console.timeEnd('cost');
})

```

运行结果

```
WarningLampPlugin
Accelerating to hello
tapPromise to ilovetapable
cost: 1003.898ms
```

#### 进阶（上面的例子与webpack的关联）

我们将刚才的代码稍作改动，拆成两个文件：Compiler.js、Myplugin.js

Compiler.js

- 把Class Car类名改成webpack的核心`Compiler`
- 接受options里传入的plugins
- 将Compiler作为参数传给plugin
- 执行run函数，在编译的每个阶段，都触发执行相对应的钩子函数。

```js
const {
    SyncHook,
    AsyncParallelHook
} = require('tapable');

class Compiler {
    constructor(options) {
        this.hooks = {
            accelerate: new SyncHook(["newSpeed"]),
            break: new SyncHook(),
            calculateRoutes: new AsyncParallelHook(["source", "target", "routesList"])
        };
        let plugins = options.plugins;
        if (plugins && plugins.length > 0) {
            // 对传递的plugin依次调用apply方法，绑定hook，apply方法接受compiler参数
            plugins.forEach(plugin => plugin.apply(this));
        }
    }
    run(){
        console.time('cost');
        this.accelerate('hello')
        this.break()
        this.calculateRoutes('i', 'like', 'tapable')
    }
    accelerate(param){
        // 在相应的生命周期调用相应的钩子
        this.hooks.accelerate.call(param);
    }
    break(){
        this.hooks.break.call();
    }
    calculateRoutes(){
        const args = Array.from(arguments)
        this.hooks.calculateRoutes.callAsync(...args, err => {
            console.timeEnd('cost');
            if (err) console.log(err)
        });
    }
}

module.exports = Compiler

```

MyPlugin.js

- 引入Compiler
- 定义一个自己的插件。
- apply方法接受 compiler参数。

> webpack 插件是一个具有 `apply` 方法的 JavaScript 对象。`apply 属性会被 webpack compiler 调用`，并且 compiler 对象可在整个编译生命周期访问。

- 给compiler上的钩子绑定方法。
- 仿照webpack规则，`向 plugins 属性传入 new 实例`。

```js
const Compiler = require('./Compiler')

class MyPlugin{
    constructor() {

    }
    apply(conpiler){//接受 compiler参数
        conpiler.hooks.break.tap("WarningLampPlugin", () => console.log('WarningLampPlugin'));
        conpiler.hooks.accelerate.tap("LoggerPlugin", newSpeed => console.log(`Accelerating to ${newSpeed}`));
        conpiler.hooks.calculateRoutes.tapAsync("calculateRoutes tapAsync", (source, target, routesList, callback) => {
            setTimeout(() => {
                console.log(`tapAsync to ${source}${target}${routesList}`)
                callback();
            }, 2000)
        });
    }
}


//这里类似于webpack.config.js的plugins配置
//向 plugins 属性传入 new 实例

const myPlugin = new MyPlugin();

const options = {
    plugins: [myPlugin]
}
let compiler = new Compiler(options)
compiler.run()

```

### webpack流程

通过上面的阅读，我们知道了如何在webapck事件流上挂载钩子。

假设现在要自定义一个插件更改最后产出资源的内容，我们应该把事件添加在哪个钩子上呢？哪一个步骤能拿到webpack编译的资源从而去修改？

所以接下来的任务是：了解webpack的流程。

![img](\images\e748c143c2474494989674e129daaa94tplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)

#### 1. webpack入口`（webpack.config.js+shell options）`

从配置文件package.json 和 Shell 语句中读取与合并参数，得出最终的参数；

> 每次在命令行输入 webpack 后，操作系统都会去调用 `./node_modules/.bin/webpack` 这个 shell 脚本。这个脚本会去调用 `./node_modules/webpack/bin/webpack.js` 并追加输入的参数，如 -p , -w 。

#### 2. 用yargs参数解析`（optimist）`

```js
yargs.parse(process.argv.slice(2), (err, argv, output) => {})
```

#### 3.webpack初始化

1）构建compiler对象

```js
let compiler = new Webpack(options)
```

[源码地址](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fwebpack%2Fwebpack-cli%2Fblob%2Fmaster%2Fbin%2Fcli.js%23L417)

（2）注册NodeEnvironmentPlugin插件

```js
new NodeEnvironmentPlugin().apply(compiler);
```

[源码地址](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fwebpack%2Fwebpack%2Fblob%2Fmaster%2Flib%2Fwebpack.js%23L41)

（3）挂在options中的基础插件，调用`WebpackOptionsApply`库初始化基础插件。

```js
if (options.plugins && Array.isArray(options.plugins)) {
	for (const plugin of options.plugins) {
		if (typeof plugin === "function") {
			plugin.apply(compiler);
		} else {
			plugin.apply(compiler);
		}
	}
}
compiler.hooks.environment.call();
compiler.hooks.afterEnvironment.call();
compiler.options = new WebpackOptionsApply().process(options, compiler);
```

[源码地址](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fwebpack%2Fwebpack%2Fblob%2Fmaster%2Flib%2Fwebpack.js%23L53)

#### 4. `run` 开始编译

```js
if (firstOptions.watch || options.watch) {
	const watchOptions = firstOptions.watchOptions || firstOptions.watch || options.watch || {};
	if (watchOptions.stdin) {
		process.stdin.on("end", function(_) {
			process.exit(); // eslint-disable-line
		});
		process.stdin.resume();
	}
	compiler.watch(watchOptions, compilerCallback);
	if (outputOptions.infoVerbosity !== "none") console.log("\nwebpack is watching the files…\n");
} else compiler.run(compilerCallback);

```

这里分为两种情况：

1）Watching：监听文件变化

2）run：执行编译

#### 5.触发`compile`

（1）在run的过程中，已经触发了一些钩子：`beforeRun->run->beforeCompile->compile->make->seal` (编写插件的时候，就可以将自定义的方挂在对应钩子上，按照编译的顺序被执行)

（2）构建了关键的 `Compilation`对象

在`run()`方法中，执行了`this.compile()`

`this.compile()`中创建了`compilation`

```js
this.hooks.beforeRun.callAsync(this, err => {
    ...
	this.hooks.run.callAsync(this, err => {
        ...
		this.readRecords(err => {
            ...
			this.compile(onCompiled);
		});
	});
});

...

compile(callback) {
	const params = this.newCompilationParams();
	this.hooks.beforeCompile.callAsync(params, err => {
		...
		this.hooks.compile.call(params);
		const compilation = this.newCompilation(params);
		this.hooks.make.callAsync(compilation, err => {
            ...
			compilation.finish();
			compilation.seal(err => {
                ...
				this.hooks.afterCompile.callAsync(compilation, err 
				    ...
					return callback(null, compilation);
				});
			});
		});
	});
}
```

```js
const compilation = this.newCompilation(params);
```

`Compilation`负责整个编译过程，包含了每个构建环节所对应的方法。对象内部保留了对compiler的引用。

当 Webpack 以开发模式运行时，每当检测到文件变化，一次新的 Compilation 将被创建。

划重点：Compilation很重要！编译生产资源变换文件都靠它。

#### 6.addEntry() `make 分析入口文件创建模块对象`

compile中触发`make`事件并调用`addEntry`

webpack的make钩子中, tapAsync注册了一个`DllEntryPlugin`, 就是将入口模块通过调用compilation。

这一注册在`Compiler.compile()`方法中被执行。

`addEntry`方法将所有的入口模块添加到编译构建队列中，开启编译流程。

DllEntryPlugin.js

```js
compiler.hooks.make.tapAsync("DllEntryPlugin", (compilation, callback) => {
	compilation.addEntry(
		this.context,
		new DllEntryDependency(
			this.entries.map((e, idx) => {
				const dep = new SingleEntryDependency(e);
				dep.loc = {
					name: this.name,
					index: idx
				};
				return dep;
			}),
			this.name
		),
		this.name,
		callback
	);
});
```

之前`WebpackOptionsApply.process()初始化插件的时候`，执行了`compiler.hooks.entryOption.call(options.context, options.entry)`;

WebpackOptionsApply.js

```js
class WebpackOptionsApply extends OptionsApply {
	process(options, compiler) {
	    ...
	    compiler.hooks.entryOption.call(options.context, options.entry);
	}
}
```



DllPlugin.js

```js
compiler.hooks.entryOption.tap("DllPlugin", (context, entry) => {
	const itemToPlugin = (item, name) => {
		if (Array.isArray(item)) {
			return new DllEntryPlugin(context, item, name);
		}
		throw new Error("DllPlugin: supply an Array as entry");
	};
	if (typeof entry === "object" && !Array.isArray(entry)) {
		Object.keys(entry).forEach(name => {
			itemToPlugin(entry[name], name).apply(compiler);
		});
	} else {
		itemToPlugin(entry, "main").apply(compiler);
	}
	return true;
});
```



#### 7. 构建模块

`compilation.addEntry`中执行 `_addModuleChain()`这个方法主要做了两件事情。一是根据模块的类型获取对应的模块工厂并创建模块，二是构建模块。

通过 `ModuleFactory.create`方法创建模块，（有NormalModule , MultiModule , ContextModule , DelegatedModule 等）对模块使用的loader进行加载。调用 acorn 解析经 loader 处理后的源文件生成抽象语法树 AST。遍历 AST，构建该模块所依赖的模块


```js
addEntry(context, entry, name, callback) {
	const slot = {
		name: name,
		request: entry.request,
		module: null
	};
	this._preparedEntrypoints.push(slot);
	this._addModuleChain(
		context,
		entry,
		module => {
			this.entries.push(module);
		},
		(err, module) => {
			if (err) {
				return callback(err);
			}

			if (module) {
				slot.module = module;
			} else {
				const idx = this._preparedEntrypoints.indexOf(slot);
				this._preparedEntrypoints.splice(idx, 1);
			}
			return callback(null, module);
		}
	);
}
```

[addEntry addModuleChain()源码地址](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fwebpack%2Fwebpack%2Fblob%2Fmaster%2Flib%2FCompilation.js%23L1072)

#### 8. 封装构建结果（seal）

webpack 会监听 seal事件调用各插件对构建后的结果进行封装，要逐次对每个 module 和 chunk 进行整理，生成编译后的源码，合并，拆分，生成 hash 。 同时这是我们在开发时进行代码优化和功能添加的关键环节。

```js
template.getRenderMainfest.render()
```

通过模板（MainTemplate、ChunkTemplate）把chunk生产`_webpack_requie()`的格式。

#### 9. 输出资源（emit）

把Assets输出到output的path中。

## 参考

https://juejin.cn/post/6949040393165996040



