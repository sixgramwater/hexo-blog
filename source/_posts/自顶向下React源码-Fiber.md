---
title: 自顶向下React源码-Fiber
date: {{ date }}
tags: React
cover: https://ppoffice.github.io/hexo-theme-icarus/gallery/covers/vector_landscape_1.svg
toc: true
excerpt: React源码中Fiber占据了重要地位，它是整个可中断渲染得以实现的基础，存储了React运行的重要信息。下面我们深入理解一下fiber结构，并同时理解fiber构造与更新的过程。
---
## Fiber原理

### 起源

> 最早的`Fiber`官方解释来源于[2016年React团队成员Acdlite的一篇介绍 (opens new window)](https://github.com/acdlite/react-fiber-architecture)。

从上一章的学习我们知道：

在`React15`及以前，`Reconciler`采用递归的方式创建虚拟DOM，递归过程是不能中断的。如果组件树的层级很深，递归会占用线程很多时间，造成卡顿。

为了解决这个问题，`React16`将**递归的无法中断的更新**重构为**异步的可中断更新**，由于曾经用于递归的**虚拟DOM**数据结构已经无法满足需要。于是，全新的`Fiber`架构应运而生

### Fiber的含义

`Fiber`包含三层含义：

1. 作为架构来说，之前`React15`的`Reconciler`采用递归的方式执行，数据保存在递归调用栈中，所以被称为`stack Reconciler`。`React16`的`Reconciler`基于`Fiber节点`实现，被称为`Fiber Reconciler`。
2. 作为**静态的数据结构**来说，每个`Fiber节点`对应一个`React element`，保存了该组件的类型（函数组件/类组件/原生组件...）、对应的DOM节点等信息。
3. 作为**动态的工作单元**来说，每个`Fiber节点`保存了本次更新中该组件改变的状态、要执行的工作（需要被删除/被插入页面中/被更新...）。

### Fiber的结构

你可以从这里看到[Fiber节点的属性定义](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactFiber.new.js#L117)。虽然属性很多，但我们可以按三层含义将他们分类来看

```js
function FiberNode(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,
) {
  // 作为静态数据结构的属性
  this.tag = tag;
  this.key = key;
  this.elementType = null;
  this.type = null;
  this.stateNode = null;

  // 用于连接其他Fiber节点形成Fiber树
  this.return = null;
  this.child = null;
  this.sibling = null;
  this.index = 0;

  this.ref = null;

  // 作为动态的工作单元的属性
  this.pendingProps = pendingProps;
  this.memoizedProps = null;
  this.updateQueue = null;
  this.memoizedState = null;
  this.dependencies = null;

  this.mode = mode;

  this.effectTag = NoEffect;
  this.nextEffect = null;

  this.firstEffect = null;
  this.lastEffect = null;

  // 调度优先级相关
  this.lanes = NoLanes;
  this.childLanes = NoLanes;

  // 指向该fiber在另一次更新时对应的fiber
  this.alternate = null;
}
```



#### 1.作为架构来说-Fiber树

多个`Fiber节点`是如何连接形成树呢？靠如下三个属性：

```js
// 指向父级Fiber节点
this.return = null;
// 指向子Fiber节点
this.child = null;
// 指向右边第一个兄弟Fiber节点
this.sibling = null;
```

<img src="https://react.iamkasong.com/img/fiber.png" alt="Fiber架构" style="zoom: 25%;" />

> 这里需要提一下，为什么父级指针叫做`return`而不是`parent`或者`father`呢？因为作为一个工作单元，`return`指节点执行完`completeWork`（本章后面会介绍）后会返回的下一个节点。子`Fiber节点`及其兄弟节点完成工作后会返回其父级节点，所以用`return`指代父级节点。



#### 2.作为静态数据结构

每个Fiber节点有个对应的`React element`, 作为一种静态的数据结构，保存了**组件相关**的信息：

```js
// Fiber对应组件的类型 Function/Class/Host...
this.tag = tag;
// key属性
this.key = key;
// 大部分情况同type，某些情况不同，比如FunctionComponent使用React.memo包裹
this.elementType = null;
// 对于 FunctionComponent，指函数本身，对于ClassComponent，指class，对于HostComponent，指DOM节点tagName
this.type = null;
// Fiber对应的真实DOM节点
this.stateNode = null;
```

#### 3.作为动态的工作单元

作为动态的工作单元，`Fiber`中如下参数保存了本次更新相关的信息，我们会在后续的更新流程中使用到具体属性时再详细介绍

```js
// 保存本次更新造成的状态改变相关信息(状态更新)
this.pendingProps = pendingProps;
this.memoizedProps = null;
this.updateQueue = null;
this.memoizedState = null;
this.dependencies = null;

this.mode = mode;

// 保存本次更新会造成的DOM操作(effect)
this.effectTag = NoEffect;
this.nextEffect = null;

this.firstEffect = null;
this.lastEffect = null;
```

如下两个字段保存调度优先级相关的信息，会在讲解`Scheduler`时介绍。

```js
// 调度优先级相关
this.lanes = NoLanes;
this.childLanes = NoLanes;
```

## Fiber的工作原理

### 双缓存fiber树

在`React`中最多会同时存在两棵`Fiber树`。当前屏幕上显示内容对应的`Fiber树`称为`current Fiber树`，正在内存中构建的`Fiber树`称为`workInProgress Fiber树`。

`current Fiber树`中的`Fiber节点`被称为`current fiber`，`workInProgress Fiber树`中的`Fiber节点`被称为`workInProgress fiber`，他们通过`alternate`属性连接。

```js
currentFiber.alternate === workInProgressFiber;
workInProgressFiber.alternate === currentFiber;
```

`React`应用的根节点通过使`current`指针在不同`Fiber树`的`rootFiber`间切换来完成`current Fiber`树指向的切换。

每次状态更新都会产生新的`workInProgress Fiber树`，通过`current`与`workInProgress`的替换，完成`DOM`更新。

### mount

考虑如下例子：

```js
function App() {
  const [num, add] = useState(0);
  return (
    <p onClick={() => add(num + 1)}>{num}</p>
  )
}

ReactDOM.render(<App/>, document.getElementById('root'));
```

1. 首次执行`ReactDOM.render`会创建`fiberRootNode`（源码中叫`fiberRoot`）和`rootFiber`。其中`fiberRootNode`是整个应用的根节点，`rootFiber`是`<App/>`所在组件树的根节点。

之所以要区分`fiberRootNode`与`rootFiber`，是因为在应用中我们可以多次调用`ReactDOM.render`渲染不同的组件树，他们会拥有不同的`rootFiber`。但是整个应用的根节点只有一个，那就是`fiberRootNode`。

`fiberRootNode`的`current`会指向当前页面上已渲染内容对应`Fiber树`，即`current Fiber树`。

<img src="https://react.iamkasong.com/img/rootfiber.png" alt="rootFiber" style="zoom:67%;" />

由于是首屏渲染，页面中还没有挂载任何`DOM`，所以`fiberRootNode.current`指向的`rootFiber`没有任何`子Fiber节点`（即`current Fiber树`为空）。



2. 接下来进入`render阶段`，根据组件返回的`JSX`在内存中依次创建`Fiber节点`并连接在一起构建`Fiber树`，被称为`workInProgress Fiber树`。（下图中右侧为内存中构建的树，左侧为页面显示的树）

在构建`workInProgress Fiber树`时会尝试复用`current Fiber树`中已有的`Fiber节点`内的属性，在`首屏渲染`时只有`rootFiber`存在对应的`current fiber`（即`rootFiber.alternate`）。

<img src="https://react.iamkasong.com/img/workInProgressFiber.png" alt="workInProgressFiber" style="zoom:50%;" />

3. 图中右侧已构建完的`workInProgress Fiber树`在`commit阶段`渲染到页面。

此时`DOM`更新为右侧树对应的样子。`fiberRootNode`的`current`指针指向`workInProgress Fiber树`使其变为`current Fiber 树`。

<img src="https://react.iamkasong.com/img/wipTreeFinish.png" alt="workInProgressFiberFinish" style="zoom:50%;" />

### Update

1. 接下来我们点击`p节点`触发状态改变，这会开启一次新的`render阶段`并构建一棵新的`workInProgress Fiber 树`。

<img src="https://react.iamkasong.com/img/wipTreeUpdate.png" alt="wipTreeUpdate" style="zoom: 50%;" />

和`mount`时一样，`workInProgress fiber`的创建可以复用`current Fiber树`对应的节点数据。

> 这个决定是否复用的过程就是Diff算法，后面章节会详细讲解

2. `workInProgress Fiber 树`在`render阶段`完成构建后进入`commit阶段`渲染到页面上。渲染完毕后，`workInProgress Fiber 树`变为`current Fiber 树`。

## 深入理解JSX

- `JSX`和`Fiber节点`是同一个东西么？
- `React Component`、`React Element`是同一个东西么，他们和`JSX`有什么关系？

### React.createElement

既然`JSX`会被编译为`React.createElement`，让我们看看他做了什么：

```js
export function createElement(type, config, children) {
  let propName;

  const props = {};

  let key = null;
  let ref = null;
  let self = null;
  let source = null;

  if (config != null) {
    // 将 config 处理后赋值给 props
    // ...省略
  }

  const childrenLength = arguments.length - 2;
  // 处理 children，会被赋值给props.children
  // ...省略

  // 处理 defaultProps
  // ...省略

  return ReactElement(
    type,
    key,
    ref,
    self,
    source,
    ReactCurrentOwner.current,
    props,
  );
}

const ReactElement = function(type, key, ref, self, source, owner, props) {
  const element = {
    // 标记这是个 React Element
    $$typeof: REACT_ELEMENT_TYPE,

    type: type,
    key: key,
    ref: ref,
    props: props,
    _owner: owner,
  };

  return element;
};
```

我们可以看到，`React.createElement`最终会调用`ReactElement`方法返回一个包含组件数据的对象，该对象有个参数`$$typeof: REACT_ELEMENT_TYPE`标记了该对象是个`React Element`。

所以调用`React.createElement`返回的对象就是`React Element`么？

`React`提供了验证合法`React Element`的全局API [React.isValidElement](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react/src/ReactElement.js#L547)，我们看下他的实现：

```js
export function isValidElement(object) {
  return (
    typeof object === 'object' &&
    object !== null &&
    object.$$typeof === REACT_ELEMENT_TYPE
  );
}
```

可以看到，`$$typeof === REACT_ELEMENT_TYPE`的非`null`对象就是一个合法的`React Element`。换言之，在`React`中，所有`JSX`在运行时的返回结果（即`React.createElement()`的返回值）都是`React Element`。

那么`JSX`和`React Component`的关系呢?

### React Component

```jsx
import React from "react";

class AppClass extends React.Component {
  render() {
    return <p>KaSong</p>;
  }
}

console.log("这是ClassComponent：", AppClass);
console.log("这是Element：", <AppClass />);

function AppFunc() {
  return <p>KaSong</p>;
}
console.log("这是FunctionComponent：", AppFunc);
console.log("这是Element：", <AppFunc />);

```

我们从下图可以看出Component与Element的不同：

![image-20220706143856071](/images/image-20220706143856071.png)

相比之下，Component像是构造函数，Element是经过构造函数实例出来的一个对象，我们可以从Element的`type`属性来找到它对应的Componnet。

我们可以从Demo控制台打印的对象看出，`ClassComponent`对应的`Element`的`type`字段为`AppClass`自身。

`FunctionComponent`对应的`Element`的`type`字段为`AppFunc`自身。

Element的结构如下所示：

```js
{
  $$typeof: Symbol(react.element),
  key: null,
  props: {},
  ref: null,
  type: ƒ AppFunc(),
  _owner: null,
  _store: {validated: false},
  _self: null,
  _source: null 
}
```

### JSX/ReactElement与Fiber对象、DOM的关系

> 这里说的JSX基本等价于ReactElement

从上面的内容我们可以发现，`JSX`是一种描述当前组件内容的数据结构，他不包含组件**schedule**、**reconcile**、**render**所需的相关信息。

比如如下信息就不包括在`JSX`中：

- 组件在更新中的`优先级`
- 组件的`state`
- 组件被打上的用于**Renderer**的`标记`

这些内容都包含在`Fiber节点`中。

所以，在组件`mount`时，`Reconciler`根据`JSX`描述的组件内容生成组件对应的`Fiber节点`。

在`update`时，`Reconciler`将`JSX`与`旧Fiber节点`保存的数据对比，生成组件对应的`新Fiber节点`，并根据对比结果为`Fiber节点`打上`标记`。

> 1. ReactElement对象: 所有采用`jsx`语法书写的节点, 都会被编译器转换, 最终会以`React.createElement(...)`的方式, 创建出来一个与之对应的`ReactElement`对象
> 2. fiber对象：`fiber对象`是通过`ReactElement`对象进行创建的, 多个`fiber对象`构成了一棵`fiber树`, `fiber树`是构造`DOM树`的数据模型, `fiber树`的任何改动, 最后都体现到`DOM树`.

#### 我们书写的 JSX 代码到 DOM 节点的转换过程:

![img](https://7kms.github.io/react-illustration-series/static/code2dom.91f1b68b.png)

> 注意:
>
> - 开发人员能够控制的是`JSX`, 也就是`ReactElement`对象.
> - `fiber树`是通过`ReactElement`生成的, 如果脱离了`ReactElement`,`fiber树`也无从谈起. 所以是`ReactElement`树(不是严格的树结构, 为了方便也称为树)驱动`fiber树`.
> - `fiber树`是`DOM树`的数据模型, `fiber树`驱动`DOM树`
>
> 开发人员通过编程只能控制`ReactElement`树的结构, `ReactElement树`驱动`fiber树`, `fiber树`再驱动`DOM树`, 最后展现到页面上. 所以`fiber树`的构造过程, 实际上就是`ReactElement`对象到`fiber`对象的转换过程.

## Render阶段-流程概览

我们接下来将深入理解fiber树构造的细节。

`Fiber节点`是如何被创建并构建`Fiber树`的呢？

`render阶段`开始于`performSyncWorkOnRoot`或`performConcurrentWorkOnRoot`方法的调用。这取决于本次更新是同步更新还是异步更新。

我们现在还不需要学习这两个方法，只需要知道在这两个方法中会调用如下两个方法：

```js
// performSyncWorkOnRoot会调用该方法
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

// performConcurrentWorkOnRoot会调用该方法
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

可以看到，他们唯一的区别是是否调用`shouldYield`。如果当前浏览器帧没有剩余时间，`shouldYield`会中止循环，直到浏览器有空闲时间后再继续遍历。

可以看到`workLoopConcurrent`相比于`Sync`, 会多一个停顿机制, 这个机制实现了`时间切片`和`可中断渲染`



结合`performUnitOfWork函数`:

```js
// ... 省略部分无关代码
function performUnitOfWork(unitOfWork: Fiber): void {
  // unitOfWork就是被传入的workInProgress
  const current = unitOfWork.alternate;
  let next;
  next = beginWork(current, unitOfWork, subtreeRenderLanes);
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // 如果没有派生出新的节点, 则进入completeWork阶段, 传入的是当前unitOfWork
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
}
```



可以明显的看出, 整个`fiber树构造`是一个深度优先遍历。

其中有 2 个重要的变量`workInProgress`和`current`(可参考前文[fiber 树构造(基础准备)](https://7kms.github.io/react-illustration-series/main/fibertree-prepare#双缓冲技术)中介绍的`双缓冲技术`):

- `workInProgress`和`current`都视为指针
- `workInProgress`指向当前正在构造的`fiber`节点
- `current = workInProgress.alternate`(即`fiber.alternate`), 指向当前页面正在使用的`fiber`节点. 初次构造时, 页面还未渲染, 此时`current = null`.

> 在这个过程中, 内存里会同时存在 2 棵`fiber树`:
>
> - 其一: 代表当前界面的`fiber`树(已经被展示出来, 挂载到`fiberRoot.current`上). 如果是初次构造(`初始化渲染`), 页面还没有渲染, 此时界面对应的 fiber 树为空(`fiberRoot.current = null`).
> - 其二: 正在构造的`fiber`树(即将展示出来, 挂载到`HostRootFiber.alternate`上, 正在构造的节点称为`workInProgress`). 当构造完成之后, 重新渲染页面, 最后切换`fiberRoot.current = workInProgress`, 使得`fiberRoot.current`重新指向代表当前界面的`fiber`树.

![image-20220706151023987](/images/image-20220706151023987.png)

在深度优先遍历中, 每个`fiber`节点都会经历 2 个阶段:

1. 探寻阶段 `beginWork`
2. 回溯阶段 `completeWork`

这 2 个阶段共同完成了每一个`fiber`节点的创建, 所有`fiber`节点则构成了`fiber树`.

### beginWork

首先从`rootFiber`开始向下深度优先遍历。为遍历到的每个`Fiber节点`调用[beginWork方法](https://github.com/facebook/react/blob/970fa122d8188bafa600e9b5214833487fbf1092/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L3058)。

该方法会根据传入的`Fiber节点`创建`子Fiber节点`，并将这两个`Fiber节点`连接起来。

当遍历到叶子节点（即没有子组件的组件）时就会进入“归”阶段



### completeWork

在“归”阶段会调用[completeWork](https://github.com/facebook/react/blob/970fa122d8188bafa600e9b5214833487fbf1092/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L652)处理`Fiber节点`。

当某个`Fiber节点`执行完`completeWork`，如果其存在`兄弟Fiber节点`（即`fiber.sibling !== null`），会进入其`兄弟Fiber`的“递”阶段。

如果不存在`兄弟Fiber`，会进入`父级Fiber`的“归”阶段。

“递”和“归”阶段会交错执行直到“归”到`rootFiber`。至此，`render阶段`的工作就结束



### Fiber树的遍历方式与顺序

```js
// packages/react-reconciler/src/ReactFiberScheduler.js
function workLoop() {
  while (workInProgress !== null) {
    workInProgress = performUnitOfWork(workInProgress);
  }
}

// packages/react-reconciler/src/ReactFiberScheduler.js
function performUnitOfWork(unitOfWork: Fiber): Fiber | null {
  let next;
  next = beginWork(current, unitOfWork, renderExpirationTime);
  if (next === null) {
    next = completeUnitOfWork(unitOfWork);
  }
  return next;
}
```

遍历需要一个指针指向当前遍历到的节点，`workInProgress` 就是这个指针，进一步是 `performUnitOfWork` 的 `next` 指针，遍历在指针为 null 的时候结束。

`next` 先从 `beginWork` 获取，如果没有，就从 `completeUnitOfWork` 获取。这里 `beginWork` 是“递”，即不停向下找到当前分支最深叶子节点的过程；`completeUnitOfWork` 是“归”，即结束这个分支，向右或向上的过程。



#### 递 - beginWork

beginWork本身对于递归没什么实际进展，主要是根据 tag 分发逻辑。根据updateXXX的不同，后续逻辑也不同。

但从遍历顺序而言，`beginWork`返回当前fiber节点的第一个子节点或者是null. 然后交给`comleteUnitOfWork`

#### 归 - completeUnitOfWork

需要注意的是，`next` 指针不应该重复经过同一个节点。因为如果向下的过程中经过某个节点，在向上的过程中又出现，就会再次进入 `beginWork`，造成死循环。继续看 `completeUnitOfWork` 如何解决这个问题。

```ts
function completeUnitOfWork(unitOfWork: Fiber): Fiber | null {
  completedWork = unitOfWork;
  do {
    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      return siblingFiber;
    }
    const returnFiber = completedWork.return;
    completedWork = returnFiber;
  } while (completedWork !== null);
  return null;
}
```

`completeUnitOfWork` 内部又创建了一层循环，搭配一个向上的新指针 `completeWork`，然后循环看当前指针节点，有**兄弟节点**就返回交还给外层循环，没有就一直向上到父节点`completedWork.return`，直到最上面的根节点。

#### 一张图总结

假设我们有如下这样一棵树。

![img](/images/v2-eabcf03d0c43acfedb92c3933575c8f9_720w.jpg)

- 整个遍历由 `performUnitOfWork` 发起，为深度优先遍历
- 从根节点开始，循环调 `beginWork` 向下爬树（黄色箭头，每个箭头表示一次调用）
- 到达叶子节点（`beginWork` 爬不下去）后，调 `completeUnitOfWork` 向上爬到下一个未遍历过的节点，也就是第一个出现的祖先兄弟节点（绿色箭头，每个箭头表示一次调用）
- 所以 `beginWork` 可能连续调用多次，一次最多只爬一步，但 `completeUnitOfWork` 只可能在 `beginWork` 之间连续调用一次，一次可以向上爬若干步
- `completeUnitOfWork` 内部包下了**若干步循环向上**的爬树操作（绿色虚线箭头）

## Render阶段-beginWork

从上一节我们已经知道，`beginWork`的工作是传入`当前Fiber节点`，创建`子Fiber节点`，我们从传参来看看具体是如何做的。

### 从传参看执行

```js
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  // ...省略函数体
}
```

其中传参：

- current：当前组件对应的`Fiber节点`在上一次更新时的`Fiber节点`，即`workInProgress.alternate`
- workInProgress：当前组件对应的`Fiber节点`
- renderLanes：优先级相关，在讲解`Scheduler`时再讲解

> 除[`rootFiber`](https://react.iamkasong.com/process/doubleBuffer.html#mount时)以外， 组件`mount`时，由于是首次渲染，是不存在当前组件对应的`Fiber节点`在上一次更新时的`Fiber节点`，即`mount`时`current === null`。
>
> 组件`update`时，由于之前已经`mount`过，所以`current !== null`。
>
> 所以我们可以通过`current === null ?`来区分组件是处于`mount`还是`update`。

基于此原因，`beginWork`的工作可以分为两部分：

- `update`时：如果`current`存在，在满足一定条件时可以复用`current`节点（**bailout逻辑**），这样就能克隆`current.child`作为`workInProgress.child`，而不需要新建`workInProgress.child`。
- `mount`时：除`fiberRootNode`以外，`current === null`。会根据`fiber.tag`不同，创建不同类型的`子Fiber节点`

```ts
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {

  // update时：如果current存在可能存在优化路径，可以复用current（即上一次更新的Fiber节点）
  if (current !== null) {
    // ...省略

    // 复用current => bailout逻辑
    return bailoutOnAlreadyFinishedWork(
      current,
      workInProgress,
      renderLanes,
    );
  } else {
    didReceiveUpdate = false;
  }

  // mount时：根据tag不同，创建不同的子Fiber节点
  switch (workInProgress.tag) {
    case IndeterminateComponent: 
      // ...省略
    case LazyComponent: 
      // ...省略
    case FunctionComponent: 
      // ...省略
    case ClassComponent: 
      // ...省略
    case HostRoot:
      // ...省略
    case HostComponent:
      // ...省略
    case HostText:
      // ...省略
    // ...省略其他类型
  }
}
```

### beginWork流程逻辑图

![image-20220706175534080](/images/image-20220706175534080.png)     



### Update时

```js
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const updateLanes = workInProgress.lanes;
  if (current !== null) {
    // 进入对比
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;
    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      didReceiveUpdate = true;
    } else if (!includesSomeLane(renderLanes, updateLanes)) {
      // 当前渲染优先级renderLanes不包括fiber.lanes, 表明当前fiber节点无需更新
      didReceiveUpdate = false;
      switch (
        workInProgress.tag
        // switch 语句中包括 context相关逻辑, 本节暂不讨论(不影响分析fiber树构造)
      ) {
      }
      // 当前fiber节点无需更新, 调用bailoutOnAlreadyFinishedWork循环检测子节点是否需要更新
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
  }
  // 余下逻辑与初次创建共用
  // 1. 设置workInProgress优先级为NoLanes(最高优先级)
  workInProgress.lanes = NoLanes;
  // 2. 根据workInProgress节点的类型, 用不同的方法派生出子节点
  switch (
    workInProgress.tag // 只列出部分case
  ) {
    case ClassComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps =
        workInProgress.elementType === Component
          ? unresolvedProps
          : resolveDefaultProps(Component, unresolvedProps);
      return updateClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    case HostRoot:
      return updateHostRoot(current, workInProgress, renderLanes);
    case HostComponent:
      return updateHostComponent(current, workInProgress, renderLanes);
    case HostText:
      return updateHostText(current, workInProgress);
    case Fragment:
      return updateFragment(current, workInProgress, renderLanes);
  }
}
```



#### bailout逻辑

https://juejin.cn/post/6907546624441090055#heading-1

> bailout的条件
>
> 
>
> 我们可以看到，满足如下情况时`didReceiveUpdate === false`（即可以直接复用前一次更新的`子Fiber`，不需要新建`子Fiber`）
>
> 1. `oldProps === newProps && workInProgress.type === current.type`，即`props`与`fiber.type`不变
> 2. `!includesSomeLane(renderLanes, updateLanes)`，即当前`Fiber节点`优先级不够，会在讲解`Scheduler`时介绍



`bailout`的优化还不止如此。如果一棵`fiber`子树所有节点都没有更新，即使所有子孙`fiber`都走`bailout`逻辑，还是有遍历的成本。

所以，在`bailout`中，会检查该`fiber`的所有子孙`fiber`是否满足条件2（该检查时间复杂度`O(1)`）。

如果所有子孙`fiber`本次都没有更新需要执行，则`bailout`会直接返回`null`。整棵子树都被跳过。

不会`bailout`也不会`render`，就像不存在一样。对应的DOM不会产生任何变化。

```js
if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      didReceiveUpdate = true;
    } else if (!includesSomeLane(renderLanes, updateLanes)) {
      didReceiveUpdate = false;
      switch (workInProgress.tag) {
        // 省略处理
      }
      return bailoutOnAlreadyFinishedWork(
        current,
        workInProgress,
        renderLanes,
      );
    } else {
      didReceiveUpdate = false;
    }
  } else {
    didReceiveUpdate = false;
  }
```

#### shouldComponentUpdate与bailout

使用`ShouldComponentUpdate`是为了减少不必要的`render`，换句话说：让本该`render`的组件走`bailout`逻辑。

当使用`shouldComponentUpdate`，这个组件`bailout`的条件会产生变化：

> -- ~~`oldProps === newProps`~~
>
> ++ `ShouldCompoentUpdate === false`



同理，使用`PureComponenet`和`React.memo`时，`bailout`的条件也会产生变化：

> -- ~~`oldProps === newProps`~~
>
> ++ `浅比较oldProps与newsProps相等`

#### 新老Context的实现对比

https://juejin.cn/post/6907546624441090055#heading-4



### Mount时

当不满足优化路径时，我们就进入第二部分，新建`子Fiber`。

我们可以看到，根据`fiber.tag`不同，进入不同类型`Fiber`的创建逻辑。

> 可以从[这里](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactWorkTags.js)看到`tag`对应的组件类型

```js
// mount时：根据tag不同，创建不同的Fiber节点
switch (workInProgress.tag) {
  case IndeterminateComponent: 
    // ...省略
  case LazyComponent: 
    // ...省略
  case FunctionComponent: 
    // ...省略
  case ClassComponent: 
    // ...省略
  case HostRoot:
    // ...省略
  case HostComponent:
    // ...省略
  case HostText:
    // ...省略
  // ...省略其他类型
}
```

对于我们常见的组件类型，如（`FunctionComponent`/`ClassComponent`/`HostComponent`），最终会进入[reconcileChildren](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L233)方法。

### updateXXX

> `updateXXX`函数(如: `updateHostRoot`, `updateClassComponent` 等)的主要逻辑:
>
> 1. 根据 `ReactElement`对象创建所有的`fiber`节点, 最终构造出`fiber树形结构`(设置`return`和`sibling`指针)
> 2. 设置`fiber.flags`(二进制形式变量, 用来标记 `fiber`节点 的`增,删,改`状态, 等待`completeWork阶段处理`)
> 3. 设置`fiber.stateNode`局部状态(如`Class类型`节点: `fiber.stateNode=new Class()`)

`updateXXX`函数(如: updateHostRoot, updateClassComponent 等)虽然 case 较多, 但是主要逻辑可以概括为 3 个步骤:

1. 根据`fiber.pendingProps, fiber.updateQueue`等`输入数据`状态, 计算`fiber.memoizedState`作为`输出状态`
2. 获取下级`ReactElement`对象
   1. class 类型的 `fiber` 节点
      - 构建`React.Component`实例
      - 把新实例挂载到`fiber.stateNode`上
      - 执行`render`之前的生命周期函数
      - 执行`render`方法, 获取下级`reactElement`
      - 根据实际情况, 设置`fiber.flags`
   2. function 类型的 `fiber` 节点
      - 执行 function, 获取下级`reactElement`
      - 根据实际情况, 设置`fiber.flags`
   3. HostComponent 类型(如: `div, span, button` 等)的 `fiber` 节点
      - `pendingProps.children`作为下级`reactElement`
      - 如果下级节点是文本节点,则设置下级节点为 null. 准备进入`completeUnitOfWork`阶段
      - 根据实际情况, 设置`fiber.flags`
   4. 其他类型
3. 根据`ReactElement`对象, 调用`reconcileChildren`生成`Fiber`子节点(只生成`次级子节点`)
   1. 根据实际情况, 设置`fiber.flags`

不同的`updateXXX`函数处理的`fiber`节点类型不同, 总的目的是为了向下生成子节点. 在这个过程中把一些需要持久化的数据挂载到`fiber`节点上(如`fiber.stateNode`,`fiber.memoizedState`等); 把`fiber`节点的特殊操作设置到`fiber.flags`(如:`节点ref`,`class组件的生命周期`,`function组件的hook`,`节点删除`等).

这里列出`updateHostRoot`, `updateHostComponent`的代码, 对于其他常用 case 的分析(如`class`类型, `function`类型), 在`状态组件`章节中进行探讨.





### reconcileChildren

从该函数名就能看出这是`Reconciler`模块的核心部分。那么他究竟做了什么呢？

> 调和函数是`updateXXX`函数中的一项重要逻辑, 它的作用是向下生成子节点, 并设置`fiber.flags`.
>
> - `mount`时`fiber`节点没有比较对象, 所以在向下生成子节点的时候没有任何多余的逻辑, 只管创建就行.
> - `update`时需要把`ReactElement`对象与`旧fiber`对象进行比较, 来判断是否需要复用`旧fiber`对象.(也就是俗称的`Diff`算法），将比较的结果生成新`Fiber节点`



```js
export function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderLanes: Lanes
) {
  if (current === null) {
    // 对于mount的组件
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderLanes,
    );
  } else {
    // 对于update的组件
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderLanes,
    );
  }
}
```

从代码可以看出，和`beginWork`一样，他也是通过`current === null ?`区分`mount`与`update`。

不论走哪个逻辑，最终他会生成新的子`Fiber节点`并赋值给`workInProgress.child`，作为本次`beginWork`[返回值](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L1158)，并作为下次`performUnitOfWork`执行时`workInProgress`的[传参](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1702)。



> 注意：值得一提的是，`mountChildFibers`与`reconcileChildFibers`这两个方法的逻辑基本一致。唯一的区别是：`reconcileChildFibers`会为生成的`Fiber节点`带上`effectTag`属性，而`mountChildFibers`不会。



### effectTag

我们知道，`render阶段`的工作是在内存中进行，当工作结束后会通知`Renderer`需要执行的`DOM`操作。要执行`DOM`操作的具体类型就保存在`fiber.effectTag`中。

> 你可以从[这里](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactSideEffectTags.js)看到`effectTag`对应的`DOM`操作

比如：

```js
// DOM需要插入到页面中
export const Placement = /*                */ 0b00000000000010;
// DOM需要更新
export const Update = /*                   */ 0b00000000000100;
// DOM需要插入到页面中并更新
export const PlacementAndUpdate = /*       */ 0b00000000000110;
// DOM需要删除
export const Deletion = /*                 */ 0b00000000001000;
```

> 通过二进制表示`effectTag`，可以方便的使用位操作为`fiber.effectTag`赋值多个`effect`。

那么，如果要通知`Renderer`将`Fiber节点`对应的`DOM节点`插入页面中，需要满足两个条件：

1. `fiber.stateNode`存在，即`Fiber节点`中保存了对应的`DOM节点`
2. `(fiber.effectTag & Placement) !== 0`，即`Fiber节点`存在`Placement effectTag`

我们知道，`mount`时，`fiber.stateNode === null`，且在`reconcileChildren`中调用的`mountChildFibers`不会为`Fiber节点`赋值`effectTag`。那么首屏渲染如何完成呢？

针对第一个问题，`fiber.stateNode`会在`completeWork`中创建，我们会在下一节介绍。

第二个问题的答案十分巧妙：假设`mountChildFibers`也会赋值`effectTag`，那么可以预见`mount`时整棵`Fiber树`所有节点都会有`Placement effectTag`。那么`commit阶段`在执行`DOM`操作时每个节点都会执行一次插入操作，这样大量的`DOM`操作是极低效的。

为了解决这个问题，在`mount`时只有`rootFiber`会赋值`Placement effectTag`，在`commit阶段`只会执行一次插入操作。

## Render阶段-completeWork

### 流程概览

#### completeUnitOfWork

`completeUnitOfWork(unitOfWork)`, 处理 `beginWork` 阶段已经创建出来的 `fiber` 节点, 核心逻辑:

1. 调用`completeWork`
   - 【mount阶段】 给`fiber`节点(tag=HostComponent, HostText)**创建 DOM 实例**, 设置`fiber.stateNode`局部状态(如`tag=HostComponent, HostText`节点: fiber.stateNode 指向这个 DOM 实例).
   - 【mount阶段】为 DOM 节点设置属性, 绑定事件(这里先说明有这个步骤, 详细的事件处理流程, 在`合成事件原理`中详细说明).
   - 设置`fiber.flags`标记
   - 【update阶段】如果 DOM 属性有变化, 不会再次新建 DOM 对象, 而是设置`fiber.flags |= Update`, 等待`commit`阶段处理
2. 把当前 `fiber` 对象的**副作用队列**(`firstEffect`和`lastEffect`)添加到父节点的副作用队列之后, 更新父节点的`firstEffect`和`lastEffect`指针.
3. 识别`beginWork`阶段设置的`fiber.flags`, 判断当前 `fiber` 是否有副作用(增,删,改), 如果有, 需要将当前 `fiber` 加入到父节点的`effects`队列, 等待`commit`阶段处理.

#### completeWork

类似`beginWork`，`completeWork`也是针对不同`fiber.tag`调用不同的处理逻辑。

```js
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
  const newProps = workInProgress.pendingProps;

  switch (workInProgress.tag) {
    case IndeterminateComponent:
    case LazyComponent:
    case SimpleMemoComponent:
    case FunctionComponent:
    case ForwardRef:
    case Fragment:
    case Mode:
    case Profiler:
    case ContextConsumer:
    case MemoComponent:
      return null;
    case ClassComponent: {
      // ...省略
      return null;
    }
    case HostRoot: {
      // ...省略
      updateHostContainer(workInProgress);
      return null;
    }
    case HostComponent: {
      // ...省略
      return null;
    }
  // ...省略
```

我们重点关注页面渲染所必须的`HostComponent`（即原生`DOM组件`对应的`Fiber节点`），其他类型`Fiber`的处理留在具体功能实现时讲解。



### completeWork流程逻辑图

![image-20220706215816487](/images/image-20220706215816487.png)

### 处理HostComponent

和`beginWork`一样，我们根据`current === null ?`判断是`mount`还是`update`。

同时针对`HostComponent`，判断`update`时我们还需要考虑`workInProgress.stateNode != null ?`（即该`Fiber节点`是否存在对应的`DOM节点`）

```js
case HostComponent: {
  popHostContext(workInProgress);
  const rootContainerInstance = getRootHostContainer();
  const type = workInProgress.type;

  if (current !== null && workInProgress.stateNode != null) {
    // update的情况
    // ...省略
  } else {
    // mount的情况
    // ...省略
  }
  return null;
}
```

### Update时

当`update`时，`Fiber节点`已经存在对应`DOM节点`，所以不需要生成`DOM节点`。需要做的主要是处理`props`，比如：

- `onClick`、`onChange`等回调函数的注册
- 处理`style prop`
- 处理`DANGEROUSLY_SET_INNER_HTML prop`
- 处理`children prop`

我们去掉一些当前不需要关注的功能（比如`ref`）。可以看到最主要的逻辑是调用`updateHostComponent`方法。

```js
if (current !== null && workInProgress.stateNode != null) {
  // update的情况
  updateHostComponent(
    current,
    workInProgress,
    type,
    newProps,
    rootContainerInstance,
  );
}
```

在`updateHostComponent`内部，被处理完的`props`会被赋值给`workInProgress.updateQueue`，并最终会在`commit阶段`被渲染在页面上。

```ts
workInProgress.updateQueue = (updatePayload: any);
```

其中`updatePayload`为数组形式，他的偶数索引的值为变化的`prop key`，奇数索引的值为变化的`prop value`。

### mount时

同样，我们省略了不相关的逻辑。可以看到，`mount`时的主要逻辑包括三个：

- 为`Fiber节点`生成对应的`DOM节点`
- 将子孙`DOM节点`插入刚生成的`DOM节点`中
- 与`update`逻辑中的`updateHostComponent`类似的处理`props`的过程

```js
// mount的情况

// ...省略服务端渲染相关逻辑

const currentHostContext = getHostContext();
// 为fiber创建对应DOM节点
const instance = createInstance(
    type,
    newProps,
    rootContainerInstance,
    currentHostContext,
    workInProgress,
  );
// 将子孙DOM节点插入刚生成的DOM节点中
appendAllChildren(instance, workInProgress, false, false);
// DOM节点赋值给fiber.stateNode
workInProgress.stateNode = instance;

// 与update逻辑中的updateHostComponent类似的处理props的过程
if (
  finalizeInitialChildren(
    instance,
    type,
    newProps,
    rootContainerInstance,
    currentHostContext,
  )
) {
  markUpdate(workInProgress);
}
```

还记得[上一节](https://react.iamkasong.com/process/beginWork.html#effecttag)我们讲到：`mount`时只会在`rootFiber`存在`Placement effectTag`。那么`commit阶段`是如何通过一次插入`DOM`操作（对应一个`Placement effectTag`）将整棵`DOM树`插入页面的呢？

原因就在于`completeWork`中的`appendAllChildren`方法。

由于`completeWork`属于“归”阶段调用的函数，每次调用`appendAllChildren`时都会将已生成的子孙`DOM节点`插入当前生成的`DOM节点`下。那么当“归”到`rootFiber`时，我们已经有一个构建好的离屏`DOM树`。

### effectList

至此`render阶段`的绝大部分工作就完成了。

还有一个问题：作为`DOM`操作的依据，`commit阶段`需要找到所有有`effectTag`的`Fiber节点`并依次执行`effectTag`对应操作。难道需要在`commit阶段`再遍历一次`Fiber树`寻找`effectTag !== null`的`Fiber节点`么？

这显然是很低效的。

为了解决这个问题，在`completeWork`的上层函数`completeUnitOfWork`中，每个执行完`completeWork`且存在`effectTag`的`Fiber节点`会被保存在一条被称为`effectList`的单向链表中。

`effectList`中第一个`Fiber节点`保存在`fiber.firstEffect`，最后一个元素保存在`fiber.lastEffect`。

类似`appendAllChildren`，在“归”阶段，所有有`effectTag`的`Fiber节点`都会被追加在`effectList`中，最终形成一条以`rootFiber.firstEffect`为起点的单向链表。

```js
                       nextEffect         nextEffect
rootFiber.firstEffect -----------> fiber -----------> fiber
```

这样，在`commit阶段`只需要遍历`effectList`就能执行所有`effect`了。

你可以在[这里 (opens new window)](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1744)看到这段代码逻辑。

> 借用`React`团队成员**Dan Abramov**的话：`effectList`相较于`Fiber树`，就像圣诞树上挂的那一串彩灯。

### 流程结尾

至此，`render阶段`全部工作完成。在`performSyncWorkOnRoot`函数中`fiberRootNode`被传递给`commitRoot`方法，开启`commit阶段`工作流程。

```js
commitRoot(root);
```





## Commit阶段-流程概览

`commitRoot`方法是`commit阶段`工作的起点。`fiberRootNode`会作为传参。

```js
commitRoot(root);
```

Render阶段对应的是fiber树的构造，Commit阶段对应的是fiber树渲染过程，实际上是对fiber树的进一步处理

通过前文`fiber树构造`的解读, 可以总结出`fiber树`的基本特点:

- 无论是`首次构造`或者是`对比更新`, 最终都会在内存中生成一棵用于渲染页面的`fiber树`(即`fiberRoot.finishedWork`).
- 这棵将要被渲染的`fiber树`有 2 个特点:
  1. 副作用队列挂载在根节点上(具体来讲是`finishedWork.firstEffect`)
  2. 代表最新页面的`DOM`对象挂载在`fiber树`中首个`HostComponent`类型的节点上(具体来讲`DOM`对象是挂载在`fiber.stateNode`属性上)



![img](/images/fibertree-beforecommit.a32107ef.png)

### 主要工作与流程

在`rootFiber.firstEffect`上保存了一条需要执行`副作用`的`Fiber节点`的单向链表`effectList`，这些`Fiber节点`的`updateQueue`中保存了变化的`props`。

这些`副作用`对应的`DOM操作`在`commit`阶段执行。

除此之外，一些生命周期钩子（比如`componentDidXXX`）、`hook`（比如`useEffect`）需要在`commit`阶段执行。

`commit`阶段的主要工作（即`Renderer`的工作流程）分为三部分：

- before mutation阶段（执行`DOM`操作前）
- mutation阶段（执行`DOM`操作）
- layout阶段（执行`DOM`操作后）

在`before mutation阶段`之前和`layout阶段`之后还有一些额外工作，涉及到比如`useEffect`的触发、`优先级相关`的重置、`ref`的绑定/解绑。

这些对我们当前属于超纲内容，为了内容完整性，在这节简单介绍

`commitRootImpl`函数中, 可以根据是否调用渲染, 把整个`commitRootImpl`分为 3 段(分别是`渲染前`, `渲染`, `渲染后`).

### 渲染前-before mutation之前

为接下来正式渲染, 做一些准备工作. 主要包括:

1. 设置全局状态(如: 更新`fiberRoot`上的属性)

2. 重置全局变量(如: `workInProgressRoot`, `workInProgress`等)

3. 再次更新副作用队列: 只针对根节点`fiberRoot.finishedWork`

   - 默认情况下根节点的副作用队列是不包括自身的, 如果根节点有副作用, 则将根节点添加到副作用队列的末尾
   - 注意只是延长了副作用队列, 但是`fiberRoot.lastEffect`指针并没有改变. 比如首次构造时, 根节点拥有`Snapshot`标记:

![img](/images/fiber-effectlist.9f96bfb9.png)

### 渲染-三个步骤

`commitRootImpl`函数中, 渲染阶段的主要逻辑是处理**副作用队列**, 将最新的 DOM 节点(**已经在内存中**, 只是还没渲染)渲染到界面上.

整个渲染过程被分为 3 个函数分布实现:

1. `commitBeforeMutationEffects`

   - dom 变更之前, 处理副作用队列中带有`Snapshot`,`Passive`标记的`fiber`节点.

2. `commitMutationEffects`
   - dom 变更, 界面得到更新. 处理副作用队列中带有`Placement`, `Update`, `Deletion`, `Hydrating`标记的`fiber`节点.
   
3. `commitLayoutEffects`
   - dom 变更后, 处理副作用队列中带有`Update | Callback`标记的`fiber`节点.

通过上述源码分析, 可以把`commitRootImpl`的职责概括为 2 个方面:

1. 处理副作用队列. (步骤 1,2,3 都会处理, 只是处理节点的标识`fiber.flags`不同).
2. 调用渲染器, 输出最终结果. (在步骤 2: `commitMutationEffects`中执行).

所以`commitRootImpl`是处理`fiberRoot.finishedWork`这棵即将被渲染的`fiber`树, 理论上无需关心这棵`fiber`树是如何产生的(可以是`首次构造`产生, 也可以是`对比更新`产生). 为了清晰简便, 在下文的所有图示都使用`初次创建的fiber树结构`来进行演示.

这 3 个函数处理的对象是`副作用队列`和`DOM对象`.

所以无论`fiber树`结构有多么复杂, 到了`commitRoot`阶段, 实际起作用的只有 2 个节点:

- `副作用队列`所在节点: 根节点, 即`HostRootFiber`节点.
- `DOM对象`所在节点: 从上至下首个`HostComponent`类型的`fiber`节点, 此节点 `fiber.stateNode`实际上指向最新的 DOM 树.

下图为了清晰, 省略了一些无关引用, 只留下`commitRoot`阶段实际会用到的`fiber`节点:

![img](https://7kms.github.io/react-illustration-series/static/fiber-noredundant.f4ab819d.png)



### 渲染后-layout以后



## Commit阶段-beforeMutation



`before mutation阶段`的代码很短，整个过程就是遍历`effectList`并调用`commitBeforeMutationEffects`函数处理。

```js
// 保存之前的优先级，以同步优先级执行，执行完毕后恢复之前优先级
const previousLanePriority = getCurrentUpdateLanePriority();
setCurrentUpdateLanePriority(SyncLanePriority);

// 将当前上下文标记为CommitContext，作为commit阶段的标志
const prevExecutionContext = executionContext;
executionContext |= CommitContext;

// 处理focus状态
focusedInstanceHandle = prepareForCommit(root.containerInfo);
shouldFireAfterActiveInstanceBlur = false;

// beforeMutation阶段的主函数
commitBeforeMutationEffects(finishedWork);

focusedInstanceHandle = null;
```

我们重点关注`beforeMutation`阶段的主函数`commitBeforeMutationEffects`做了什么。

### commitBeforeMutationEffects

大体代码逻辑：

```js
function commitBeforeMutationEffects() {
  while (nextEffect !== null) {
    const current = nextEffect.alternate;

    if (!shouldFireAfterActiveInstanceBlur && focusedInstanceHandle !== null) {
      // ...focus blur相关
    }

    const effectTag = nextEffect.effectTag;

    // 调用getSnapshotBeforeUpdate
    if ((effectTag & Snapshot) !== NoEffect) {
      commitBeforeMutationEffectOnFiber(current, nextEffect);
    }

    // 调度useEffect
    if ((effectTag & Passive) !== NoEffect) {
      if (!rootDoesHavePassiveEffects) {
        rootDoesHavePassiveEffects = true;
        scheduleCallback(NormalSchedulerPriority, () => {
          flushPassiveEffects();
          return null;
        });
      }
    }
    nextEffect = nextEffect.nextEffect;
  }
}
```

>  整体可以分为三部分：
>
> 1. 处理`DOM节点`渲染/删除后的 `autoFocus`、`blur` 逻辑。
> 2. 调用`getSnapshotBeforeUpdate`生命周期钩子。
> 3. 调度`useEffect`。

我们讲解下2、3两点。

### 调用getSnapshotBeforeUpdate

`commitBeforeMutationEffectOnFiber`是`commitBeforeMutationLifeCycles`的别名。

在该方法内会调用`getSnapshotBeforeUpdate`。

从`React`v16开始，`componentWillXXX`钩子前增加了`UNSAFE_`前缀。

究其原因，是因为`Stack Reconciler`重构为`Fiber Reconciler`后，`render阶段`的任务可能中断/重新开始，对应的组件在`render阶段`的生命周期钩子（即`componentWillXXX`）***<u>可能触发多次</u>***。

这种行为和`React`v15不一致，所以标记为`UNSAFE_`。

> 更详细的解释参照[这里](https://juejin.im/post/6847902224287285255#comment)

为此，`React`提供了替代的生命周期钩子`getSnapshotBeforeUpdate`。

我们可以看见，`getSnapshotBeforeUpdate`是在`commit阶段`内的`before mutation阶段`调用的，由于`commit阶段`是**同步**的，所以**不会遇到多次调用**的问题。

### 调度`useEffect`

在这几行代码内，`scheduleCallback`方法由`Scheduler`模块提供，用于以某个优先级异步调度一个回调函数。

```js
// 调度useEffect
if ((effectTag & Passive) !== NoEffect) {
  if (!rootDoesHavePassiveEffects) {
    rootDoesHavePassiveEffects = true;
    scheduleCallback(NormalSchedulerPriority, () => {
      // 触发useEffect
      flushPassiveEffects();
      return null;
    });
  }
}
```

在此处，被异步调度的回调函数就是触发`useEffect`的方法`flushPassiveEffects`。

我们接下来讨论`useEffect`如何被异步调度，以及**为什么要*<u>异步</u>*（而不是同步）调度**。

#### 如何异步调度

在`flushPassiveEffects`方法内部会从全局变量`rootWithPendingPassiveEffects`获取`effectList`。

关于`flushPassiveEffects`的具体讲解参照[useEffect与useLayoutEffect一节](https://react.iamkasong.com/hooks/useeffect.html)

在[completeWork一节](https://react.iamkasong.com/process/completeWork.html#effectlist)我们讲到，`effectList`中保存了需要执行副作用的`Fiber节点`。其中副作用包括

- 插入`DOM节点`（Placement）
- 更新`DOM节点`（Update）
- 删除`DOM节点`（Deletion）

除此外，当一个`FunctionComponent`含有`useEffect`或`useLayoutEffect`，他对应的`Fiber节点`也会被赋值`effectTag`。

> 你可以从[这里](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactHookEffectTags.js)看到`hook`相关的`effectTag`

在`flushPassiveEffects`方法内部会遍历`rootWithPendingPassiveEffects`（即`effectList`）执行`effect`回调函数。

如果在此时直接执行，`rootWithPendingPassiveEffects === null`。

那么`rootWithPendingPassiveEffects`会在何时赋值呢？

在上一节`layout之后`的代码片段中会根据`rootDoesHavePassiveEffects === true?`决定是否赋值`rootWithPendingPassiveEffects`。

```js
const rootDidHavePassiveEffects = rootDoesHavePassiveEffects;
if (rootDoesHavePassiveEffects) {
  rootDoesHavePassiveEffects = false;
  rootWithPendingPassiveEffects = root;
  pendingPassiveEffectsLanes = lanes;
  pendingPassiveEffectsRenderPriority = renderPriorityLevel;
}
```

所以整个`useEffect`异步调用分为三步：

1. `before mutation阶段`在`scheduleCallback`中调度`flushPassiveEffects`
2. `layout阶段`之后将`effectList`赋值给`rootWithPendingPassiveEffects`
3. `scheduleCallback`触发`flushPassiveEffects`，`flushPassiveEffects`内部遍历`rootWithPendingPassiveEffects`

#### 为什么需要异步调用

摘录自`React`文档[effect 的执行时机](https://zh-hans.reactjs.org/docs/hooks-reference.html#timing-of-effects)：

> 与 `componentDidMount`、`componentDidUpdate` 不同的是，在浏览器完成布局与绘制之后，传给 `useEffect` 的函数会**延迟调用**。这使得它适用于许多常见的副作用场景，比如设置订阅和事件处理等情况，因此不应在函数中执行阻塞浏览器更新屏幕的操作。

可见，`useEffect`异步执行的原因主要是**防止同步执行时阻塞浏览器渲染**。

## Commit阶段-mutation

### 概览

类似`before mutation阶段`，`mutation阶段`也是遍历`effectList`，依次执行`commitMutationEffects`。该方法的主要工作为“根据`effectTag`调用不同的处理函数处理`Fiber`。

```js
nextEffect = firstEffect;
do {
  try {
      commitMutationEffects(root, renderPriorityLevel);
    } catch (error) {
      invariant(nextEffect !== null, 'Should be working on an effect.');
      captureCommitPhaseError(nextEffect, error);
      nextEffect = nextEffect.nextEffect;
    }
} while (nextEffect !== null);
```

### commitMutationEffects

```js
function commitMutationEffects(root: FiberRoot, renderPriorityLevel) {
  // 遍历effectList
  while (nextEffect !== null) {

    const effectTag = nextEffect.effectTag;

    // 根据 ContentReset effectTag重置文字节点
    if (effectTag & ContentReset) {
      commitResetTextContent(nextEffect);
    }

    // 更新ref
    if (effectTag & Ref) {
      const current = nextEffect.alternate;
      if (current !== null) {
        commitDetachRef(current);
      }
    }

    // 根据 effectTag 分别处理
    const primaryEffectTag =
      effectTag & (Placement | Update | Deletion | Hydrating);
    switch (primaryEffectTag) {
      // 插入DOM
      case Placement: {
        commitPlacement(nextEffect);
        nextEffect.effectTag &= ~Placement;
        break;
      }
      // 插入DOM 并 更新DOM
      case PlacementAndUpdate: {
        // 插入
        commitPlacement(nextEffect);

        nextEffect.effectTag &= ~Placement;

        // 更新
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      // SSR
      case Hydrating: {
        nextEffect.effectTag &= ~Hydrating;
        break;
      }
      // SSR
      case HydratingAndUpdate: {
        nextEffect.effectTag &= ~Hydrating;

        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      // 更新DOM
      case Update: {
        const current = nextEffect.alternate;
        commitWork(current, nextEffect);
        break;
      }
      // 删除DOM
      case Deletion: {
        commitDeletion(root, nextEffect, renderPriorityLevel);
        break;
      }
    }

    nextEffect = nextEffect.nextEffect;
  }
}
```

`commitMutationEffects`会遍历`effectList`，对每个`Fiber节点`执行如下三个操作：

1. 根据`ContentReset effectTag`重置文字节点
2. 更新`ref`
3. 根据`effectTag`分别处理，其中`effectTag`包括(`Placement` | `Update` | `Deletion` | `Hydrating`)

我们关注步骤三中的`Placement` | `Update` | `Deletion`。`Hydrating`作为服务端渲染相关，我们先不关注。

### Placement effect

当`Fiber节点`含有`Placement effectTag`，意味着该`Fiber节点`对应的`DOM节点`需要插入到页面中。

调用的方法为`commitPlacement`。

该方法所做的工作分为三步：

1. 获取父级`DOM节点`。其中`finishedWork`为传入的`Fiber节点`。

```js
const parentFiber = getHostParentFiber(finishedWork);
// 父级DOM节点
const parentStateNode = parentFiber.stateNode;
```

1. 获取`Fiber节点`的`DOM`兄弟节点

```js
const before = getHostSibling(finishedWork);
```

1. 根据`DOM`兄弟节点是否存在决定调用`parentNode.insertBefore`或`parentNode.appendChild`执行`DOM`插入操作。

```js
// parentStateNode是否是rootFiber
if (isContainer) {
  insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent);
} else {
  insertOrAppendPlacementNode(finishedWork, before, parent);
}
```



值得注意的是，`getHostSibling`（获取兄弟`DOM节点`）的执行很耗时，当在同一个父`Fiber节点`下依次执行多个插入操作，`getHostSibling`算法的复杂度为指数级。

这是由于`Fiber节点`不只包括`HostComponent`，所以`Fiber树`和渲染的`DOM树`节点并不是一一对应的。要从`Fiber节点`找到`DOM节点`很可能跨层级遍历。



### Update effect

当`Fiber节点`含有`Update effectTag`，意味着该`Fiber节点`需要更新。调用的方法为`commitWork`，他会根据`Fiber.tag`分别处理。

这里我们主要关注`FunctionComponent`和`HostComponent`。

#### FunctionComponent mutation

当`fiber.tag`为`FunctionComponent`，会调用`commitHookEffectListUnmount`。该方法会遍历`effectList`，执行所有`useLayoutEffect hook`的销毁函数。

#### HostComponent mutation

当`fiber.tag`为`HostComponent`，会调用`commitUpdate`。

最终会在[`updateDOMProperties` (opens new window)](https://github.com/facebook/react/blob/970fa122d8188bafa600e9b5214833487fbf1092/packages/react-dom/src/client/ReactDOMComponent.js#L378)中将[`render阶段 completeWork` (opens new window)](https://github.com/facebook/react/blob/970fa122d8188bafa600e9b5214833487fbf1092/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L229)中为`Fiber节点`赋值的`updateQueue`对应的内容渲染在页面上。

```js
for (let i = 0; i < updatePayload.length; i += 2) {
  const propKey = updatePayload[i];
  const propValue = updatePayload[i + 1];

  // 处理 style
  if (propKey === STYLE) {
    setValueForStyles(domElement, propValue);
  // 处理 DANGEROUSLY_SET_INNER_HTML
  } else if (propKey === DANGEROUSLY_SET_INNER_HTML) {
    setInnerHTML(domElement, propValue);
  // 处理 children
  } else if (propKey === CHILDREN) {
    setTextContent(domElement, propValue);
  } else {
  // 处理剩余 props
    setValueForProperty(domElement, propKey, propValue, isCustomComponentTag);
  }
}
```



### Deletion effect

当`Fiber节点`含有`Deletion effectTag`，意味着该`Fiber节点`对应的`DOM节点`需要从页面中删除。调用的方法为`commitDeletion`。

该方法会执行如下操作：

1. 递归调用`Fiber节点`及其子孙`Fiber节点`中`fiber.tag`为`ClassComponent`的[`componentWillUnmount` (opens new window)](https://github.com/facebook/react/blob/970fa122d8188bafa600e9b5214833487fbf1092/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L920)生命周期钩子，从页面移除`Fiber节点`对应`DOM节点`
2. 解绑`ref`
3. 调度`useEffect`的销毁函数

## Commit阶段-Layout

该阶段之所以称为`layout`，因为该阶段的代码都是在`DOM`渲染完成（`mutation阶段`完成）后执行的。

该阶段触发的生命周期钩子和`hook`可以直接访问到已经改变后的`DOM`，即该阶段是可以参与`DOM layout`的阶段。

### 概览

与前两个阶段类似，`layout阶段`也是遍历`effectList`，执行函数。

具体执行的函数是`commitLayoutEffects`。

```js
root.current = finishedWork;

nextEffect = firstEffect;
do {
  try {
    commitLayoutEffects(root, lanes);
  } catch (error) {
    invariant(nextEffect !== null, "Should be working on an effect.");
    captureCommitPhaseError(nextEffect, error);
    nextEffect = nextEffect.nextEffect;
  }
} while (nextEffect !== null);

nextEffect = null;
```

### commitLayoutEffects

```js
function commitLayoutEffects(root: FiberRoot, committedLanes: Lanes) {
  while (nextEffect !== null) {
    const effectTag = nextEffect.effectTag;

    // 调用生命周期钩子和hook
    if (effectTag & (Update | Callback)) {
      const current = nextEffect.alternate;
      commitLayoutEffectOnFiber(root, current, nextEffect, committedLanes);
    }

    // 赋值ref
    if (effectTag & Ref) {
      commitAttachRef(nextEffect);
    }

    nextEffect = nextEffect.nextEffect;
  }
}
```

`commitLayoutEffects`一共做了两件事：

1. commitLayoutEffectOnFiber（调用`生命周期钩子`和`hook`相关操作）
2. commitAttachRef（赋值 ref）

### commitLayoutEffectOnFiber

`commitLayoutEffectOnFiber`方法会根据`fiber.tag`对不同类型的节点分别处理

- 对于`ClassComponent`，他会通过`current === null?`区分是`mount`还是`update`，调用[`componentDidMount` ](https://github.com/facebook/react/blob/970fa122d8188bafa600e9b5214833487fbf1092/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L538)或[`componentDidUpdate`](https://github.com/facebook/react/blob/970fa122d8188bafa600e9b5214833487fbf1092/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L592)。

触发`状态更新`的`this.setState`如果赋值了第二个参数`回调函数`，也会在此时调用。

```js
this.setState({ xxx: 1 }, () => {
  console.log("i am update~");
});
```

- 对于`FunctionComponent`及相关类型，他会调用`useLayoutEffect hook`的`回调函数`，调度`useEffect`的`销毁`与`回调`函数

> 相关类型`指特殊处理后的`FunctionComponent`，比如`ForwardRef`、`React.memo`包裹的`FunctionComponent

```js
  switch (finishedWork.tag) {
    // 以下都是FunctionComponent及相关类型
    case FunctionComponent:
    case ForwardRef:
    case SimpleMemoComponent:
    case Block: {
      // 执行useLayoutEffect的回调函数
      commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);
      // 调度useEffect的销毁函数与回调函数
      schedulePassiveEffects(finishedWork);
      return;
    }
```

在上一节介绍[Update effect](https://react.iamkasong.com/renderer/mutation.html#update-effect)时介绍过，`mutation阶段`会执行`useLayoutEffect hook`的`销毁函数`。

结合这里我们可以发现，`useLayoutEffect hook`从上一次更新的`销毁函数`调用到本次更新的`回调函数`调用是同步执行的。

而`useEffect`则需要先调度，在`Layout阶段`完成后再异步执行。

这就是`useLayoutEffect`与`useEffect`的区别。

- 对于`HostRoot`，即`rootFiber`，如果赋值了第三个参数`回调函数`，也会在此时调用。

```js
ReactDOM.render(<App />, document.querySelector("#root"), function() {
  console.log("i am mount~");
});
```
