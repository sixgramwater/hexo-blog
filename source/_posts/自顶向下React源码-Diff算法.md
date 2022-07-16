---
title: 自顶向下React源码-Diff
date: {{ date }}
tags: React
cover: https://ppoffice.github.io/hexo-theme-icarus/gallery/covers/vector_landscape_3.svg
toc: true
excerpt: React源码中Diff算法到底是什么？我们为什么需要diff算法？它什么时候发生，又是具体怎么运作的？这些问题需要我们深入探究React render阶段，并理解React Diff算法的精髓，此外我们还会探究React与Vue Diff算法的不同，进行比较分析他们的优劣。
---
## 概览

### React Diff的特性

算法复杂度低, 从上至下比较整个树形结构, 时间复杂度被缩短到 O(n)

### Diff的瓶颈以及React如何应对

由于`Diff`操作本身也会带来性能损耗，React文档中提到，即使在最前沿的算法中，将前后两棵树完全比对的算法的复杂程度为 O(n 3 )，其中`n`是树中元素的数量。

如果在`React`中使用了该算法，那么展示1000个元素所需要执行的计算量将在十亿的量级范围。这个开销实在是太过高昂。

为了降低算法复杂度，`React`的`diff`会预设三个限制：

1. 只对同级元素进行`Diff`。如果一个`DOM节点`在前后两次更新中跨越了层级，那么`React`不会尝试复用他。
2. 两个不同类型的元素会产生出不同的树。如果元素由`div`变为`p`，React会销毁`div`及其子孙节点，并新建`p`及其子孙节点。
3. 开发者可以通过 `key prop`来暗示哪些子元素在不同的渲染下能保持稳定。考虑如下例子：

```jsx
// 更新前
<div>
  <p key="ka">ka</p>
  <h3 key="song">song</h3>
</div>

// 更新后
<div>
  <h3 key="song">song</h3>
  <p key="ka">ka</p>
</div>
```

如果没有`key`，`React`会认为`div`的第一个子节点由`p`变为`h3`，第二个子节点由`h3`变为`p`。这符合限制2的设定，会销毁并新建。

但是当我们用`key`指明了节点前后对应关系后，`React`知道`key === "ka"`的`p`在更新后还存在，所以`DOM节点`可以复用，只是需要交换下顺序。

这就是`React`为了应对算法性能瓶颈做出的三条限制。

### 基本原理

1. 比较对象:`fiber`对象与`ReactElement`对象相比较.
   - 注意: 此处有一个误区, 并<u>不是两棵 fiber 树相比较</u>, 而是`旧fiber`对象与`新ReactElement`对象向比较, 结果生成新的`fiber子节点`.
   - 可以理解为输入`ReactElement`, 经过`reconcileChildren()`之后, 输出`fiber`.
2. 比较方案:
   - 单节点比较
   - 可迭代节点比较

### Diff是如何实现的

我们从`Diff`的入口函数`reconcileChildFibers`出发，该函数会根据`newChild`（即`JSX对象`）类型调用不同的处理函数。

```js
// 根据newChild类型选择不同diff函数处理
function reconcileChildFibers(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChild: any,
): Fiber | null {

  const isObject = typeof newChild === 'object' && newChild !== null;

  if (isObject) {
    // object类型，可能是 REACT_ELEMENT_TYPE 或 REACT_PORTAL_TYPE
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        // 调用 reconcileSingleElement 处理
      // // ...省略其他case
    }
  }

  if (typeof newChild === 'string' || typeof newChild === 'number') {
    // 调用 reconcileSingleTextNode 处理
    // ...省略
  }

  if (isArray(newChild)) {
    // 调用 reconcileChildrenArray 处理
    // ...省略
  }

  // 一些其他情况调用处理函数
  // ...省略

  // 以上都没有命中，删除节点
  return deleteRemainingChildren(returnFiber, currentFirstChild);
}
```

我们可以从同级的节点数量将Diff分为两类：

1. 当`newChild`类型为`object`、`number`、`string`，代表同级只有一个节点
2. 当`newChild`类型为`Array`，同级有多个节点

在接下来两节我们会分别讨论这两类节点的`Diff`。

## 单节点Diff

对于单个节点，我们以类型`object`为例，会进入`reconcileSingleElement`

> 你可以从[这里](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactChildFiber.new.js#L1141)看到`reconcileSingleElement`源码

```javascript
  const isObject = typeof newChild === 'object' && newChild !== null;

  if (isObject) {
    // 对象类型，可能是 REACT_ELEMENT_TYPE 或 REACT_PORTAL_TYPE
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        // 调用 reconcileSingleElement 处理
      // ...其他case
    }
  }
```

```js
// 只保留主干逻辑
function reconcileSingleElement(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  element: ReactElement,
  lanes: Lanes,
): Fiber {
  const key = element.key;
  let child = currentFirstChild;

  while (child !== null) {
    // currentFirstChild !== null, 表明是对比更新阶段
    if (child.key === key) {
      // 1. key相同, 进一步判断 child.elementType === element.type
      switch (child.tag) {
        // 只看核心逻辑
        default: {
          if (child.elementType === element.type) {
            // 1.1 已经匹配上了, 如果有兄弟节点, 需要给兄弟节点打上Deletion标记
            deleteRemainingChildren(returnFiber, child.sibling);
            // 1.2 构造fiber节点, 新的fiber对象会复用current.stateNode, 即可复用DOM对象
            const existing = useFiber(child, element.props);
            existing.ref = coerceRef(returnFiber, child, element);
            existing.return = returnFiber;
            return existing;
          }
          break;
        }
      }
      // Didn't match. 给当前节点点打上Deletion标记
      deleteRemainingChildren(returnFiber, child);
      break;
    } else {
      // 2. key不相同, 匹配失败, 给当前节点打上Deletion标记
      deleteChild(returnFiber, child);
    }
    child = child.sibling;
  }

  {
    // ...省略部分代码, 只看核心逻辑
  }

  // 新建节点
  const created = createFiberFromElement(element, returnFiber.mode, lanes);
  created.ref = coerceRef(returnFiber, currentFirstChild, element);
  created.return = returnFiber;
  return created;
}
```

1. 如果是新增节点, 直接新建 fiber, 没有多余的逻辑
2. 如果是对比更新
   - 如果`key`和`type`都相同(即: `ReactElement.key` === `Fiber.key` 且 `Fiber.elementType === ReactElement.type`), 则复用
   - 否则新建

注意: 复用过程是调用`useFiber(child, element.props)`创建`新的fiber`对象, 这个`新fiber对象.stateNode = currentFirstChild.stateNode`, 即`stateNode`属性得到了复用, 故 DOM 节点得到了复用.



## 多节点Diff

上一节我们介绍了单一节点的`Diff`，现在考虑我们有一个`FunctionComponent`：

```jsx
function List () {
  return (
    <ul>
      <li key="0">0</li>
      <li key="1">1</li>
      <li key="2">2</li>
      <li key="3">3</li>
    </ul>
  )
}
```

他的返回值`JSX对象`的`children`属性不是单一节点，而是包含四个对象的数组

```js
{
  $$typeof: Symbol(react.element),
  key: null,
  props: {
    children: [
      {$$typeof: Symbol(react.element), type: "li", key: "0", ref: null, props: {…}, …}
      {$$typeof: Symbol(react.element), type: "li", key: "1", ref: null, props: {…}, …}
      {$$typeof: Symbol(react.element), type: "li", key: "2", ref: null, props: {…}, …}
      {$$typeof: Symbol(react.element), type: "li", key: "3", ref: null, props: {…}, …}
    ]
  },
  ref: null,
  type: "ul"
}
```

这种情况下，`reconcileChildFibers`的`newChild`参数类型为`Array`，在`reconcileChildFibers`函数内部对应如下情况：

```js
  if (isArray(newChild)) {
    // 调用 reconcileChildrenArray 处理
    // ...省略
  }
```

这一节我们来看看，如何处理同级多个节点的`Diff`。



### 概览

首先归纳下我们需要处理的情况：

1. 节点更新
2. 节点新增或减少
3. 节点位置变化

同级多个节点的`Diff`，一定属于以上三种情况中的一种或多种。

### reconcileChildrenArray

`reconcileChildrenArray`函数源码看似很长, 梳理其主干之后, 其实非常清晰.

通过形参, 首先明确比较对象是`currentFirstChild: Fiber | null`和`newChildren: Array<*>`:

- `currentFirstChild`: 是一个`fiber`节点, 通过`fiber.sibling`可以将兄弟节点全部遍历出来. 所以可以将`currentFirstChild`理解为链表头部, 它代表一个序列, 源码中被记为`oldFiber`.
- `newChildren`: 是一个数组, 其中包含了若干个`ReactElement`对象. 所以`newChildren`也代表一个序列.

所以`reconcileChildrenArray`实际就是 2 个序列之间的比较(`链表oldFiber`和`数组newChildren`), 最后返回合理的`fiber`序列.

上述代码中, 以注释分割线为界限, 整个核心逻辑分为 2 步骤:

1. 第一次循环: 遍历最长`公共`序列(key 相同), 公共序列的节点都视为可复用
   - 如果`newChildren序列`被遍历完, 那么`oldFiber序列`中剩余节点都视为**删除**(打上`Deletion`标记)
   - 如果`oldFiber序列`被遍历完, 那么`newChildren序列`中剩余节点都视为**新增**(打上`Placement`标记)
2. 第二次循环: 遍历剩余`非公共`序列, 优先复用 oldFiber 序列中的节点
   - 在对比更新阶段(非初次创建`fiber`, 此时`shouldTrackSideEffects`被设置为true). 第二次循环遍历完成之后, `oldFiber序列中`没有匹配上的节点都视为删除(打上`Deletion`标记)

### 第一轮遍历

第一轮遍历步骤如下：

1. `let i = 0`，遍历`newChildren`，将`newChildren[i]`与`oldFiber`比较，判断`DOM节点`是否可复用。

2. 如果可复用，`i++`，继续比较`newChildren[i]`与`oldFiber.sibling`，可以复用则继续遍历。

3. 如果不可复用，分两种情况：

   - `key`不同导致不可复用，立即跳出整个遍历，**第一轮遍历结束。**

   - `key`相同`type`不同导致不可复用，会将`oldFiber`标记为`DELETION`，并继续遍历

4. 如果`newChildren`遍历完（即`i === newChildren.length - 1`）或者`oldFiber`遍历完（即`oldFiber.sibling === null`），跳出遍历，**第一轮遍历结束。**



当遍历结束后，会有两种结果：

#### 1. 步骤3跳出遍历

此时`newChildren`没有遍历完，`oldFiber`也没有遍历完。

#### 2. 步骤4跳出遍历

可能`newChildren`遍历完，或`oldFiber`遍历完，或他们同时遍历完。

### 第二轮遍历

对于第一轮遍历的结果，我们分别讨论：

#### `newChildren`与`oldFiber`同时遍历完

那就是最理想的情况：只需在第一轮遍历进行组件[`更新`](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/react-reconciler/src/ReactChildFiber.new.js#L825)。此时`Diff`结束。

#### `newChildren`没遍历完，`oldFiber`遍历完

已有的`DOM节点`都复用了，这时还有新加入的节点，意味着本次更新有新节点插入，我们只需要遍历剩下的`newChildren`为生成的`workInProgress fiber`依次标记`Placement`。

#### `newChildren`遍历完，`oldFiber`没遍历完

意味着本次更新比之前的节点数量少，有节点被删除了。所以需要遍历剩下的`oldFiber`，依次标记`Deletion`。

#### `newChildren`与`oldFiber`都没遍历完

这意味着有节点在这次更新中**改变了位置**。

这是`Diff算法`最精髓也是最难懂的部分。我们接下来会重点讲解。



### 处理移动的节点

由于有节点改变了位置，所以不能再用位置索引`i`对比前后的节点，那么如何才能将同一个节点在两次更新中对应上呢？

我们需要使用`key`。

为了快速的找到`key`对应的`oldFiber`，我们将所有还未处理的`oldFiber`存入以`key`为key，`oldFiber`为value的`Map`中。

```javascript
const existingChildren = mapRemainingChildren(returnFiber, oldFiber);
```

接下来遍历剩余的`newChildren`，通过`newChildren[i].key`就能在`existingChildren`中找到`key`相同的`oldFiber`。



### 标记节点是否移动

首先我们需要知道一个原则：React Diff算法只会计算应该将哪些节点**右移**。

然后我们需要两个变量：

- `lastPlacedIndex`用来表示当前我们已经确定的最后一个可复用节点在`oldFiber`中的位置。
- `oldIndex`当前我们对`newChildren`进行遍历时，新节点在`oldFiber`中的位置。

有两种情况：

1. `oldIndex < lastPlacedIndex`: 该节点出现在我们已经确定可复用节点的左边，因此需要**右移**
2. `oldIndex >= lastPlacedIndex`:不需要右移，同时更新`lastPlacedIndex`为`oldIndex`。



既然我们的目标是寻找移动的节点，那么我们需要明确：节点是否移动是以什么为参照物？

我们的参照物是：最后一个可复用的节点在`oldFiber`中的位置索引（用变量`lastPlacedIndex`表示）。

由于本次更新中节点是按`newChildren`的顺序排列。在遍历`newChildren`过程中，每个`遍历到的可复用节点`一定是当前遍历到的`所有可复用节点`中**最靠右的那个**，即一定在`lastPlacedIndex`对应的`可复用的节点`在本次更新中位置的后面。

那么我们只需要比较`遍历到的可复用节点`在上次更新时是否也在`lastPlacedIndex`对应的`oldFiber`后面，就能知道两次更新中这两个节点的相对位置改变没有。

我们用变量`oldIndex`表示`遍历到的可复用节点`在`oldFiber`中的位置索引。如果`oldIndex < lastPlacedIndex`，代表本次更新该节点需要向右移动。

`lastPlacedIndex`初始为`0`，每遍历一个可复用的节点，如果`oldIndex >= lastPlacedIndex`，则`lastPlacedIndex = oldIndex`。

> 第一轮遍历



### React Diff算法的不足

考虑如下情况：

```
// 之前
abcd

// 之后
dabc
```

可以看到，我们以为从 `abcd` 变为 `dabc`，只需要将`d`移动到前面。

但实际上React保持`d`不变，将`abc`分别移动到了`d`的后面。

这是由于React只能识别应该将哪些节点**右移**,因此它选择将abc右移，而不会出现将d左移的操作。



## 为什么 React 的 Diff 算法不采用 Vue 的双端对比算法？

```js
function reconcileChildrenArray(
returnFiber: Fiber,
 currentFirstChild: Fiber | null,
 newChildren: Array<*>,
 expirationTime: ExpirationTime,
): Fiber | null {
    // This algorithm can't optimize by searching from boths ends since we
    // don't have backpointers on fibers. I'm trying to see how far we can get
    // with that model. If it ends up not being worth the tradeoffs, we can
    // add it later.

    // Even with a two ended optimization, we'd want to optimize for the case
    // where there are few changes and brute force the comparison instead of
    // going for the Map. It'd like to explore hitting that path first in
    // forward-only mode and only go for the Map once we notice that we need
    // lots of look ahead. This doesn't handle reversal as well as two ended
    // search but that's unusual. Besides, for the two ended optimization to
    // work on Iterables, we'd need to copy the whole set.

    // In this first iteration, we'll just live with hitting the bad case
    // (adding everything to a Map) in for every insert/move.

    // If you change this code, also update reconcileChildrenIterator() which
    // uses the same algorithm.
｝
```

> 由于当前fiber节点是一个单链表的架构，没有反向指针，因此没有使用双端搜索算法进行优化。
>
> 

### Vue3 diff

### Vue2 diff

Vue2 的 Diff 算法就是以新的虚拟DOM为准进行与老虚拟DOM的比对，继而进行各种情况的处理。大概可以分为 4 种情况：更新节点、新增节点、删除节点、移动节点位置。比对新老两个虚拟DOM，就是通过循环，每循环到一个新节点，就去老节点列表里面找到和当前新节点相同的旧节点。如果在旧节点列表中找不到，说明当前节点是需要新增的节点，我们就需要进行创建节点并插入视图的操作；如果找到了，就做更新操作；如果找到的旧节点与新节点位置不同，则需要移动节点等。


