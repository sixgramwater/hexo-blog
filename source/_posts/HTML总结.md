---
title: HTML 总结
date: {{ date }}
tags: html
cover: https://ppoffice.github.io/hexo-theme-icarus/gallery/covers/vector_landscape_2.svg
excerpt: A total review of HTML
toc: true
---
# HTML head总结

## meta

```html

<meta charset="UTF-8" />
    <title>哔哩哔哩 (゜-゜)つロ 干杯~-bilibili</title>
    <meta
      name="description"
      content="bilibili是国内知名的视频弹幕网站，这里有及时的动漫新番，活跃的ACG氛围，有创意的Up主。大家可以在这里找到许多欢乐。"
    />
    <meta
      name="keywords"
      content="Bilibili,哔哩哔哩,哔哩哔哩动画,哔哩哔哩弹幕网...后面还有很多"
    />
    <meta name="renderer" content="webkit" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="spm_prefix" content="333.1007" />
    <meta name="referrer" content="no-referrer-when-downgrade" />
    <meta name="applicable-device" content="pc">
    <meta http-equiv="Cache-Control" content="no-transform" />
    <meta http-equiv="Cache-Control" content="no-siteapp" />
```

### meta标签常用设置

```html
<!-- 根据浏览器的屏幕大小自适应的展现合适的效果 -->
<meta name="applicable-device" content="pc,mobile" />

<!-- 移动端 浏览器中页面将以原始大小显示，不允许缩放 -->
<meta name="viewport" content="width=device-width,initial-scale=1.0,minimum-scale=1.0,maximum-scale=1.0,user-scalable=no" />

<!--自动选择更好的浏览器-->
<meta name="renderer" content="webkit">
//告诉浏览器这个网址应该用哪个内核渲染，那么浏览器就会在读取到这个标签后，立即切换对应的内核

<!-- 优先使用 IE 最新版本和 Chrome -->
<meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1" />
//它必须显示在网页中除 title 元素和其他 meta 元素以外的所有其他元素之前。如果不是的话，它不起作用

<!-- 描述文档类型 content表示文档类型，这里为text/html，如果JS就是text/javascript，charset表示页面字符集 -->
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />

<!-- iphone会把一串数字识别为电话号码，点击的时候会提示是否呼叫，屏蔽这功能则把telephone设置为no -->
<meta content="telephone=no" name="format-detection" />

<!-- iphone的私有标签，默认值为default（白色），可以定为black（黑色）和black-translucent（灰色半透明） -->
<meta content="black" name="apple-mobile-web-app-status-bar-style">

<!-- iphone设备的是有标签 允许全屏模式浏览，隐藏浏览器导航栏 -->
<meta content="yes" name="apple-mobile-web-app-capable" />

<!-- 全屏显示 -->
<meta content="yes" name="apple-touch-fullscreen" />

<!-- UC强制全屏 -->
<meta name="full-screen" content="yes">

<!-- QQ强制全屏 -->
<meta name="x5-fullscreen" content="true">

<!-- 屏蔽百度转码 -->
<meta http-equiv="Cache-Control" content="no-transform" />
<meta http-equiv="Cache-Control" content="no-siteapp" />

<!-- 定义网页简短描述 -->
<meta name="description" content="Cochemist">

<!-- 定义网页关键词 -->
<meta name="keywords" content="生物化学"> 

<!-- 定义网页的作者 -->
<meta name="author" content="sun_Annie">

<!-- 避免HTML页面缓存 -->
<meta http-equiv="pragma" content="no-cache" />
<meta http-equiv="cache-control" content="no-cache">
<meta http-equiv="expires" content="0">

<!-- 定义网页的缓存过期时间 -->
<meta http-equiv="expires" content="Sunday 26 October 2016 00:00 GMT">
//由于这是一个过去的日期，所以这个网页只要一打开，就会直接到网站服务器重新下载页面内容，而不是从cache调用。这是一种防止网页被cache缓存的措施。

```

## link

```html
<link rel="dns-prefetch" href="//s1.hdslb.com" />
<link rel="apple-touch-icon" href="//static.hdslb.com/mobile/img/512.png" />
<link rel="shortcut icon" href="https://www.bilibili.com/favicon.ico?v=1" />
<link rel="canonical" href="https://www.bilibili.com/" />
<link rel="alternate" media="only screen and(max-width: 640px)" href="https://m.bilibili.com" />
<link
      rel="stylesheet"
      href="//s1.hdslb.com/bfs/static/jinkela/long/font/medium.css"
      media="print"
      onload="this.media='all'"
/>
<link
      rel="stylesheet"
      href="//s1.hdslb.com/bfs/static/jinkela/long/font/regular.css"
      media="print"
      onload="this.media='all'"
/>
<link rel="stylesheet" href="//s1.hdslb.com/bfs/static/jinkela/long/laputa-css/map.css"/>
<link rel="stylesheet" href="//s1.hdslb.com/bfs/static/jinkela/long/laputa-css/light_u.css"/>
<link id="__css-map__" rel="stylesheet" href="//s1.hdslb.com/bfs/static/jinkela/long/laputa-css/light.css"/>
  
```

### 基于link的性能优化

link rel 的属性值可以为**preload**、**prefetch**、**dns-prefesh**

```html
<link rel="preload" href="style.css" as="style">
<link rel="prefetch" href="/uploads/images/pic.png">
<link rel="dns-prefetch" href="//fonts.googleapis.com">
```

#### preload

preload能够让你在你的HTML页面中元素内部书写一些声明式的资源获取请求，可以指明哪些资源是在页面加载完成后即刻需要的。对于这种即刻需要的资源，你可能希望在页面加载的生命周期的**早期**阶段就开始获取，在浏览器的主渲染机制介入前就进行预加载。这一机制使得资源可以更早的得到加载并可用，且更不易阻塞页面的初步渲染，进而提升性能。

简单来说，就是通过标签显式声明一个高优先级资源，强制浏览器提前请求资源，同时不阻塞文档正常onload。

除了rel,还需要提供href，as这两个属性的值

```html
<link rel="preload" href="style.css" as="style">
<link rel="preload" href="main.js" as="script">
```

##### **1. 举例**

![img](\images\80e08544d9f34106b69503b547d62a52tplv-k3u1fbpfcp-zoom-in-crop-mark1304000.awebp)

> 上图是我们开发的另外一个收银台，出于本地化的考虑，设计上使用了自定义字体。开发完成后我们发现，页面首次加载时文字会出现短暂的**字体样式闪动**（FOUT，Flash of Unstyled Text），在网络情况较差时比较明显（如动图所示）。究其原因，是**字体文件由css引入**，在**css解析后**才会进行加载，加载完成之前浏览器只能使用降级字体。也就是说，**字体文件加载的时机太迟**，需要告诉浏览器提前进行加载，这恰恰是preload的用武之地。

```html
<head>
    ...
    <link rel="preload" as="font" href="<%= require('/assets/fonts/AvenirNextLTPro-Demi.otf') %>" crossorigin>
    <link rel="preload" as="font" href="<%= require('/assets/fonts/AvenirNextLTPro-Regular.otf') %>" crossorigin>
    ...
</head>
```



#### prefetch

用户在使用当前界面时，浏览器空闲时先把下个界面要使用的资源下载到本地缓存。浏览器下不下载不可知。
举个例子： 网站有A，B 两个界面。当用户访问界面 A 时有很大的概率会访问 B 界面（比如登录跳转）那么我们可以在用户访问界面 A 的时候，提前将 B 界面需要的某些资源下载到本地，性能会得到很大的提升。那么我们只需要在界面 A.html 文件中设置如下代码：

```html
<link rel="prefetch" href="pageB/images/B.png">
```

#### dns-prefetch

当浏览器需要到其他域名请求资源时，会先进行 DNS 解析，dns-prefetch就是预先把 DNS 解析做好，当请求这些第三方域名下的资源时，可以达到一个更快速的体验

```html
<link rel="dns-prefetch" href="//s1.hdslb.com" />
```

### prefetch & preload 对比

- 关注的页面不同： prefetch 关注未来要使用的页面，preload 关注当前页面
- 资源是否下载：prefetch 的资源下载情况未知（只在浏览器空闲时会下载），preload 肯定下载

#### webpack插件：preload-webpack-plugin

前文中我们举的两个例子，都是在入口html手动添加相关代码

这显然不够方便，而且将资源路径硬编码在了页面中（实际上，ticket_bg.a5bb7c33.png后缀中的hash是构建过程自动生成的，所以硬编码的方式很多场景下本身就行不通）。webpack插件preload-webpack-plugin可以帮助我们将该过程自动化，结合htmlWebpackPlugin在构建过程中插入link标签。

```js
const PreloadWebpackPlugin = require('preload-webpack-plugin');
...
plugins: [
  new PreloadWebpackPlugin({
    rel: 'preload'，
    as(entry) {  //资源类型
      if (/\.css$/.test(entry)) return 'style';
      if (/\.woff$/.test(entry)) return 'font';
      if (/\.png$/.test(entry)) return 'image';
      return 'script';
    },
    include: 'asyncChunks', // preload模块范围，还可取值'initial'|'allChunks'|'allAssets',
    fileBlacklist: [/\.svg/] // 资源黑名单
    fileWhitelist: [/\.script/] // 资源白名单
  })
]
```

#### 使用场景

从前文的介绍可知，preload的设计初衷是为了尽早加载首屏需要的关键资源，从而提升页面渲染性能。

目前浏览器基本上都具备预测解析能力，可以提前解析入口html中外链的资源，因此入口脚本文件、样式文件等不需要特意进行preload。

但是一些**隐藏在CSS和JavaScript中的资源**，如**字体**文件，本身是首屏关键资源，但**当css文件解析之后才会被浏览器加载**。这种场景适合使用preload进行声明，尽早进行资源加载，避免页面渲染延迟。

与preload不同，prefetch声明的是将来可能访问的资源，因此适合对异步加载的模块、可能跳转到的其他路由页面进行资源缓存；对于一些将来大概率会访问的资源，如上文案例中优惠券列表的背景图、常见的加载失败icon等，也较为适用。

#### 最佳实践

基于上面对使用场景的分享，我们可以总结出一个比较通用的最佳实践：

> - 大部分场景下无需特意使用preload
> - 类似**字体**文件这种隐藏在脚本、样式中的首屏关键资源，建议使用preload
> - 异步加载的模块（典型的如单页系统中的非首页）建议使用prefetch
> - 大概率即将被访问到的资源可以使用prefetch提升性能和体验

#### 总结与踩坑

1、preload和prefetch的本质都是预加载，即先加载、后执行，加载与执行解耦。

2、preload和prefetch不会阻塞页面的onload。

3、preload用来声明当前页面的关键资源，强制浏览器尽快加载；而prefetch用来声明将来可能用到的资源，在浏览器空闲时进行加载。

4、不要滥用preload和prefetch，需要在合适的场景中使用。

5、preload的字体资源必须设置crossorigin属性，否则会导致重复加载。

原因是如果不指定crossorigin属性(即使同源)，浏览器会采用匿名模式的CORS去preload，导致两次请求无法共用缓存。

6、关于preload和prefetch资源的缓存，在Google开发者的一篇文章中是这样说明的：如果资源可以被缓存（比如说存在有效的cache-control和max-age），它被存储在HTTP缓存（也就是disk cache)中，可以被现在或将来的任务使用；如果资源不能被缓存在HTTP缓存中，作为代替，它被放在内存缓存中直到被使用。然而我们在Chrome浏览器（版本号80）中进行测试，结果却并非如此。将服务器的缓存策略设置为no-store，观察下资源加载情况。

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a85d488397e4b65abf13ec00760c1df~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

可以发现ticket_bg.png第二次加载并未从本地缓存获取，仍然是从服务器加载。因此，如果要使用prefetch，相应的资源必须做好合理的缓存控制。

7、没有合法https证书的站点无法使用prefetch，预提取的资源不会被缓存（实际使用过程中发现，原因未知）。

8、最后我们来看下preload和prefetch的浏览器兼容性。可以看到，两者的兼容性目前都还不是太好。好在不支持preload和prefetch的浏览器会自动忽略它，因此可以将它们作为一种渐进增强功能，优化我们页面的资源加载，提升性能和用户体验。

### link与@import区别

@import是CSS提供的语法规则，只能用来加载css。

@import一定要写在除@charset外的其他任何 CSS 规则之前，如果置于其它位置将会被浏览器忽略。而且，在@import之后如果存在其它样式，则@import之后的分号是必须书写，不可省略的。

#### 1.从属关系区别

`@import`是 CSS 提供的语法规则，只有导入样式表的作用；`link`是HTML提供的标签，不仅可以加载 CSS 文件，还可以定义 RSS、rel 连接属性等。

#### 2.加载顺序区别 （最重要的区别!）

加载页面时，`link`标签引入的 CSS 能够被浏览器并行（Parallel）下载（如果有多个css文件的话）；

`@import`引入的 CSS 只有在引用它的那个css文件被下载、解析之后，浏览器才会知道还有另外一个css需要下载，这时才去下载，然后下载后开始解析、构建render tree等一系列操作。 浏览器在页面所有css下载并解析完成后才会开始渲染页面。

因此css `@import`引入的css解析延迟会加长页面留白期。 所以，要尽量避免使用css `@import`而尽量采用`link`标签的方式引入。

从页面加载速度的角度考虑的话，我们应该永远使用`link`，因为`@import`阻止了浏览器对资源的并发下载。

> From a page speed standpoint, `@import` from a CSS file should almost never be used, as it can prevent stylesheets from being downloaded concurrently



#### 3.兼容性区别

`@import`是 CSS2.1 才有的语法，故只可在 IE5+ 才能识别；`link`标签作为 HTML 元素，不存在兼容性问题。

#### 4.DOM可控性区别

可以通过 JS 操作 DOM ，插入`link`标签来改变样式；由于DOM方法是基于文档的，无法使用`@import`的方式插入样式。

#### 5.权重区别

> `link`引入的样式权重大于`@import`引入的样式。（？？？）

我们在网上搜索关于这两者的区别的时候通常会看见有第5条这么一个说法。难道第5条真的是这样吗？

css的优先级表现在，对同一个html元素设置元素的时候，不同选择器的优先级不同，优先级低的样式将会被优先极高的样式所覆盖。

css的权重优先级表现为：

**`!important > 行内样式 > ID > 类、伪类、属性 > 标签名 > 继承 > 通配符`**

存疑：https://juejin.cn/post/6844903581649207309



## script

### Script标签加载与执行时机

https://juejin.cn/post/7041484566510436382

浏览器[构建`DOM`树过程](https://link.segmentfault.com/?enc=LNWkHNuE9whRrvy%2FPE%2BHrw%3D%3D.9qJJiocI69%2BrsHxhjWzmueqTEH8RNMYhwxvAvEeNBdFxEPEiSdFORMq5hzqRhUFfevgAvMiiqW%2B6V%2FwdLaYbq2K2sMManlfmieaDNvp7vWI%3D)中遇到`script`标签就会暂定构建，并开始下载脚本，执行脚本，之后再继续构建`DOM`树。

#### 预加载扫描器

> 浏览器构建DOM树时，这个过程占用了主线程。当这种情况发生时，预加载扫描仪将解析可用的内容并请求高优先级资源，如CSS、JavaScript和web字体。多亏了预加载扫描器，我们不必等到解析器找到对外部资源的引用来请求它。

浏览器并不会只有当解析到script标签的时候才会开始加载该脚本，而是会

#### 普通的script标签 - 同步加载

首先简单说明一下这里的同步加载的概念，主要含义是其**会阻塞HTML的继续解析**。我们知道HTML解析过程是从上到下解析的，而遇到需要同步加载的script标签的时候，会等待script加载完成，并执行完成后，再继续解析HTML。

那这里也简单写个示例看看是否符合预期：

```html
<script src="/1.normal1.js?wait=1000"></script>
<script src="/2.normal2.js?wait=500"></script>
<script src="/3.normal3.js?wait=500"></script>
```

其执行结果也很简单：

```
1.normal1.js wait: 1000
2.normal2.js wait: 500
3.normal3.js wait: 500
DOMContentLoaded
loaded
```

按照顺序执行，并且在`DOMContentLoaded`之前都执行完毕了，那么来看下其加载时机：

![1638408890508.jpg](\images\3de0b9f9f2a749c888f7a9cb5f4e79e3tplv-k3u1fbpfcp-zoom-in-crop-mark3024000.awebp)

你会发现，这3个脚本的加载时机竟然都是一样的，都是在HTML获取到之后，马上就开始加载。这里其实涉及到浏览器的一个优化，**它会预先找到HTML中的所有script标签，并直接开始加载，并不是在解析到该标签时才开始加载**。这样算是浏览器一种优化，等HTML解析到对应标签后，JS已经加载好了，直接执行即可，这样可以节省大量的时间，提升用户体验。当然，从上面的执行结果看，其执行时机还是保持原有逻辑的。

### async

script标签也支持`async`和`defer`属性来异步加载，即不阻塞HTML的解析。

对于async，先偷懒引用一段MDN上的介绍

> For classic scripts, if the `async` attribute is present, then the classic script will be fetched in parallel to parsing and evaluated as soon as it is available.
>
> For [module scripts](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.mozilla.org%2Fen-US%2Fdocs%2FWeb%2FJavaScript%2FGuide%2FModules), if the `async` attribute is present then the scripts and all their dependencies will be executed in the defer queue, therefore they will get fetched in parallel to parsing and evaluated as soon as they are available.
>
> This attribute allows the elimination of **parser-blocking JavaScript** where the browser would have to load and evaluate scripts before continuing to parse. `defer` has a similar effect in this case.

对于我们现在要研究的，简单来说就是**异步加载，加载完了就立即执行**。

**下载**：后台并行下载（异步下载）
**执行**：下载完就执行了。无需等待其他脚本，即先加载完的先执行。
`async`之间，以及和`DOMContentLoaded`事件处理函数执行顺序是无序的。

### defer

**下载**：后台并行下载（异步下载）
**执行**：`DOM`构建后，但在`DOMContentLoaded`事件触发前
多个`defer`按照书写顺序依次串行方式执行。
`defer`不会阻塞`DOM`树构建，但是会阻塞`DOMContentLoaded`事件触发。

### async, defer对比

**相同点**

1. 都不会阻塞`DOM`树构建，都是异步下载，可以让用户先看到页面内容；
2. 都不可以调用`document.write`，否则浏览器有个warning:

> Failed to execute 'write' on 'Document': It isn't possible to write into a document from an asynchronously-loaded external script unless it is explicitly opened.

**差异点**

1. `defer`和`async`的唯一区别就是执行时间点不同。
   - `defer`指推迟执行（执行顺序不变）
   - `async`指异步执行（执行顺序不固定）
2. 兼容性不同，defer兼容性要更好一点，async兼容性最差

### defer与将script放到body最后面有什么区别

目前没查到，后者兼容性更好。

### 如果同时添加async,defer呢？

1. 按照`async`效果执行；
2. 因为`async`是HTML5才提出的，而`defer`是HTML4就有了，所以`defer`比`async`有更好的兼容性。同时添加`async`, `defer`的另一个效果是如果浏览器不支持`async`，则降级为`defer`；
3. 最佳实践就是把 `script`标签放到`body`标签最后，防止浏览器不支持`async`，也不支持`defer`。

#### 总结

| 加载形式                      | 浏览器优化加载 | 同步 | 有序 | HTML解析后可用 |
| ----------------------------- | -------------- | ---- | ---- | -------------- |
| 普通`Script`标签              | ✅              | ✅    | ✅    | ———            |
| 普通`Script`标签async         | ✅              | ❌    | ❌    | ———            |
| 普通`Script`标签defer         | ✅              | ❌    | ✅    | ———            |
| `createElement`               | ❌              | ❌    | ❌    | ✅              |
| `createElement`且async为false | ❌              | ❌    | ✅    | ✅              |
| `document.write`普通          | ❌              | ✅    | ✅    | ❌              |
| `document.write`普通async     | ❌              | ❌    | ❌    | ❌              |
| `document.write`普通defer     | ❌              | ❌    | ✅    | ❌              |



