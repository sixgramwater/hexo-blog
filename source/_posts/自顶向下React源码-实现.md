---
title: 自顶向下React源码-实现
date: {{ date }}
tags: React
cover: https://ppoffice.github.io/hexo-theme-icarus/gallery/covers/vector_landscape_1.svg
excerpt: 主要探究React源码中如何实现状态更新的原理，什么时候会触发更新，他的调用链路是什么，update对象又是如何影响更新的
toc: true
---
## 状态更新-流程概览

### 流程图

![img](/images/workloop.66f39102.png)

### 几个关键节点

#### render阶段的开始

`render阶段`开始于`performSyncWorkOnRoot`或`performConcurrentWorkOnRoot`方法的调用。这取决于本次更新是同步更新还是异步更新。

#### commit阶段的开始

`commit阶段`开始于`commitRoot`方法的调用。其中`rootFiber`会作为传参。

我们已经知道，`render阶段`完成后会进入`commit阶段`。让我们继续补全从`触发状态更新`到`render阶段`的路径。
<!-- more -->

```sh
触发状态更新（根据场景调用不同方法）

    |
    |
    v

    ？

    |
    |
    v

render阶段（`performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot`）

    |
    |
    v

commit阶段（`commitRoot`）
```

### 调用路径

#### 创建Update对象

在`React`中，有如下方法可以触发状态更新（排除`SSR`相关）：

- ReactDOM.render
- this.setState
- this.forceUpdate
- useState
- useReducer

这些方法调用的场景各不相同，他们是如何接入同一套**状态更新机制**呢？

答案是：每次`状态更新`都会创建一个保存**更新状态相关内容**的对象，我们叫他`Update`。在`render阶段`的`beginWork`中会根据`Update`计算新的`state`。

#### 从fiber到root

现在`触发状态更新的fiber`上已经包含`Update`对象。

我们知道，`render阶段`是从`rootFiber`开始向下遍历。那么如何从`触发状态更新的fiber`得到`rootFiber`呢？

答案是：调用`markUpdateLaneFromFiberToRoot`方法。

该方法做的工作可以概括为：从`触发状态更新的fiber`一直向上遍历到`rootFiber`，并返回`rootFiber`。

由于不同更新优先级不尽相同，所以过程中还会更新遍历到的`fiber`的优先级。这对于我们当前属于超纲内容。

#### 调度更新

现在我们拥有一个`rootFiber`，该`rootFiber`对应的`Fiber树`中某个`Fiber节点`包含一个`Update`。

接下来通知`Scheduler`根据**更新**的优先级，决定以**同步**还是**异步**的方式调度本次更新。

这里调用的方法是`ensureRootIsScheduled`。



其中，`scheduleCallback`和`scheduleSyncCallback`会调用`Scheduler`提供的调度方法根据`优先级`调度回调函数执行。

可以看到，这里调度的回调函数为：

```js
performSyncWorkOnRoot.bind(null, root);
performConcurrentWorkOnRoot.bind(null, root);
```

即`render阶段`的入口函数。

#### 总结

让我们梳理下`状态更新`的整个调用路径的关键节点：

```sh
触发状态更新（根据场景调用不同方法）

    |
    |
    v

创建Update对象（接下来三节详解）

    |
    |
    v

从fiber到root（`markUpdateLaneFromFiberToRoot`）

    |
    |
    v

调度更新（`ensureRootIsScheduled`）

    |
    |
    v

render阶段（`performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot`）

    |
    |
    v

commit阶段（`commitRoot`）
```

`git pull` 就是 git fetch 和 git merge 的缩写！

## 状态更新-心智模型

### 同步更新

我们可以将`更新机制`类比`代码版本控制`。

在没有`代码版本控制`前，我们在代码中逐步叠加功能。一切看起来井然有序，直到我们遇到了一个紧急线上bug（红色节点）。

<img src="https://react.iamkasong.com/img/git1.png" alt="流程1" style="zoom: 50%;" />

为了修复这个bug，我们需要首先将之前的代码提交。

在`React`中，所有通过`ReactDOM.render`创建的应用（其他创建应用的方式参考[ReactDOM.render一节](https://react.iamkasong.com/state/reactdom.html#react的其他入口函数)）都是通过类似的方式`更新状态`。

即没有`优先级`概念，`高优更新`（红色节点）需要排在其他`更新`后面执行。

### 并发更新-Concurrent

当有了`代码版本控制`，有紧急线上bug需要修复时，我们暂存当前分支的修改，在`master分支`修复bug并紧急上线。

bug修复上线后通过`git rebase`命令和`开发分支`连接上。`开发分支`基于`修复bug的版本`继续开发。

<img src="/images/image-20220706100446067.png" alt="image-20220706100446067" style="zoom: 80%;" />

在`React`中，通过`ReactDOM.createBlockingRoot`和`ReactDOM.createRoot`创建的应用会采用`并发`的方式`更新状态`。

`高优更新`（红色节点）中断正在进行中的`低优更新`（蓝色节点），先完成`render - commit流程`。

待`高优更新`完成后，`低优更新`基于`高优更新`的结果`重新更新`。

<img src="https://react.iamkasong.com/img/git2.png" alt="流程2" style="zoom: 50%;" />

执行所谓的`git rebase D` 变基操作以后：

<img src="https://react.iamkasong.com/img/git3.png" alt="流程3" style="zoom:50%;" />

## 状态更新-Update

> 可以将`Update`类比`心智模型`中的一次`commit`。

### Update的分类

存在两种不同的Update，分别对应HostRoot、ClassComponent和FunctionComponent的更新

首先，我们将可以触发更新的方法所隶属的组件分类：

- ReactDOM.render —— HostRoot
- this.setState —— ClassComponent
- this.forceUpdate —— ClassComponent
- useState —— FunctionComponent
- useReducer —— FunctionComponent

可以看到，一共三种组件（`HostRoot` | `ClassComponent` | `FunctionComponent`）可以触发更新。

由于不同类型组件工作方式不同，所以存在**两种**不同结构的`Update`，其中`ClassComponent`与`HostRoot`共用一套`Update`结构，`FunctionComponent`单独使用一种`Update`结构。

虽然他们的结构不同，但是他们工作机制与工作流程大体相同。在本节我们介绍前一种`Update`，`FunctionComponent`对应的`Update`在`Hooks`章节介绍。

### Update的结构