---
title: React-redux原理
date: {{ date }}
tags: 
 - react
 - react-redux
cover: https://ppoffice.github.io/hexo-theme-icarus/gallery/covers/vector_landscape_1.svg
excerpt: 单纯的redux只是一个状态管理库而已，react-redux才是连接状态与视图的关键桥梁。本文我们将深入探究react-redux原理，理解provider, connect的实现。此外有必要理解zombie children现象，并理解react-redux是如何解决这一问题的。最后，react-redux在v7版本基于hooks的新版本也有必要了解。
toc: true
---
## demo
```jsx
import React from 'react';
import { connect } from 'react-redux';
import { increment, decrement, reset } from './actions';

function Counter(props) {
  const { 
    count,
    incrementHandler,
    decrementHandler,
    resetHandler
   } = props;

  return (
    <>
      <h3>Count: {count}</h3>
      <button onClick={incrementHandler}>计数+1</button>
      <button onClick={decrementHandler}>计数-1</button>
      <button onClick={resetHandler}>重置</button>
    </>
  );
}

const mapStateToProps = (state) => {
  return {
    count: state.count
  }
}

const mapDispatchToProps = (dispatch) => {
  return {
    incrementHandler: () => dispatch(increment()),
    decrementHandler: () => dispatch(decrement()),
    resetHandler: () => dispatch(reset()),
  }
};

export default connect(
  mapStateToProps,
  mapDispatchToProps
)(Counter)
```
上面代码可以看到`connect`是一个高阶函数，他的第一阶会接收`mapStateToProps`和`mapDispatchToProps`两个参数，这两个参数都是函数。`mapStateToProps`可以自定义需要将哪些state连接到当前组件，这些自定义的state可以在组件里面通过props拿到。`mapDispatchToProps`方法会传入dispatch函数，我们可以自定义一些方法，这些方法可以调用dispatch去dispatch action，从而触发state的更新，这些自定义的方法也可以通过组件的props拿到，connect的第二阶接收的参数是一个组件，我们可以猜测这个函数的作用就是将前面自定义的state和方法注入到这个组件里面，同时要返回一个新的组件给外部调用，所以`connect`其实也是一个高阶组件。

React-Redux核心其实就两个API:
- `Provider`: 用来包裹根组件的组件，作用是注入Redux的store。
- `connect`：用来将state和dispatch注入给需要的组件，返回一个新组件，他其实是个高阶组件。

所以React-Redux核心其实就两个API，而且两个都是组件，作用还很类似，都是往组件里面注入参数，`Provider`是往根组件注入store，`connect`是往需要的组件注入state和dispatch。

## Provider
```js
// Context.js
import React from 'react';

const ReactReduxContext = React.createContext();

export default ReactReduxContext;
```

然后将这个context应用到我们的Provider组件里面：

```jsx
import React from 'react';
import ReactReduxContext from './Context';

function Provider(props) {
  const {store, children} = props;

  // 这是要传递的context
  const contextValue = { store };

  // 返回ReactReduxContext包裹的组件，传入contextValue
  // 里面的内容就直接是children，我们不动他
  return (
    <ReactReduxContext.Provider value={contextValue}>
      {children}
    </ReactReduxContext.Provider>
  )
}
```

## connect
`connect`才是React-Redux中最难的部分，里面功能复杂，考虑的因素很多，想要把它搞明白我们需要一层一层的来看，首先我们实现一个只具有基本功能的`connect`。

```jsx
import React, { useContext } from 'react';
import ReactReduxContext from './Context';

// 第一层函数接收mapStateToProps和mapDispatchToProps
function connect(mapStateToProps, mapDispatchToProps) {
  // 第二层函数是个高阶组件，里面获取context
  // 然后执行mapStateToProps和mapDispatchToProps
  // 再将这个结果组合用户的参数作为最终参数渲染WrappedComponent
  // WrappedComponent就是我们使用connext包裹的自己的组件
  return function connectHOC(WrappedComponent) {

    function ConnectFunction(props) {
      // 复制一份props到wrapperProps
      const { ...wrapperProps } = props;

      // 获取context的值
      const context = useContext(ReactReduxContext);

      const { store } = context;  // 解构出store
      const state = store.getState();   // 拿到state

      // 执行mapStateToProps和mapDispatchToProps
      const stateProps = mapStateToProps(state);
      const dispatchProps = mapDispatchToProps(store.dispatch);

      // 组装最终的props
      const actualChildProps = Object.assign({}, stateProps, dispatchProps, wrapperProps);

      // 渲染WrappedComponent
      return <WrappedComponent {...actualChildProps}></WrappedComponent>
    }

    return ConnectFunction;
  }
}

export default connect;
```

### 触发更新
用上面的`Provider`和`connect`替换官方的react-redux其实已经可以渲染出页面了，但是点击按钮还不会有反应，因为我们虽然通过dispatch改变了store中的state，但是**这种改变并没有触发我们组件的更新**。

之前Redux那篇文章讲过，可以用`store.subscribe`来监听state的变化并执行回调，我们这里需要注册的回调是检查我们最终给`WrappedComponent`的props有没有变化，如果有变化就重新渲染`ConnectFunction`，所以这里我们需要解决两个问题：

1. 当我们state变化的时候检查最终给到`ConnectFunction`的参数有没有变化
1. 如果这个参数有变化，我们需要重新渲染`ConnectFunction`

#### 检查参数变化
要检查参数的变化，我们需要知道上次渲染的参数和本地渲染的参数，然后拿过来比一下就知道了。为了知道上次渲染的参数，我们可以直接在`ConnectFunction`里面使用`useRef`**将上次渲染的参数记录下来**：
```js
// 记录上次渲染参数
const lastChildProps = useRef();
useLayoutEffect(() => {
  lastChildProps.current = actualChildProps;
}, []);
```

注意`lastChildProps.current`是在第一次渲染结束后赋值，而且需要使用`useLayoutEffect`来保证渲染后立即同步执行。

因为我们检测参数变化是需要重新计算`actualChildProps`，计算的逻辑其实都是一样的，我们将这块计算逻辑抽出来，成为一个单独的方法`childPropsSelector`:

```js
// 计算actualChildProps
function childPropsSelector(store, wrapperProps) {
  const state = store.getState();   // 拿到state

  // 执行mapStateToProps和mapDispatchToProps
  const stateProps = mapStateToProps(state);
  const dispatchProps = mapDispatchToProps(store.dispatch);

  return Object.assign({}, stateProps, dispatchProps, wrapperProps);
}
```

然后就是注册store的回调，在里面来检测参数是否变了，如果变了就强制更新当前组件，对比两个对象是否相等，React-Redux里面是采用的shallowEqual，也就是浅比较，也就是只对比一层，如果你mapStateToProps返回了好几层结构，比如这样：

```js
{
  stateA: {
    value: 1
  }
}
```
你去改了stateA.value是不会触发重新渲染的

在回调里面检测参数变化
```js
// 注册回调
store.subscribe(() => {
  const newChildProps = childPropsSelector(store, wrapperProps);
  // 如果参数变了，记录新的值到lastChildProps上
  // 并且强制更新当前组件
  if(!shallowEqual(newChildProps, lastChildProps.current)) {
    lastChildProps.current = newChildProps;

    // 需要一个API来强制更新当前组件
  }
});
```

#### 强制更新
要强制更新当前组件的方法不止一个，如果你是用的Class组件，你可以直接`this.setState({})`，老版的React-Redux就是这么干的。但是新版React-Redux**用hook重写了**，那我们可以用React提供的`useReducer`或者`useStatehook`，React-Redux源码用了`useReducer`，为了跟他保持一致，我也使用`useReducer`:

```js
function storeStateUpdatesReducer(count) {
  return count + 1;
}

// ConnectFunction里面
function ConnectFunction(props) {
  // ... 前面省略n行代码 ... 
  
  // 使用useReducer触发强制更新
  const [
    ,
    forceComponentUpdateDispatch
  ] = useReducer(storeStateUpdatesReducer, 0);
  // 注册回调
  store.subscribe(() => {
    const newChildProps = childPropsSelector(store, wrapperProps);
    if(!shallowEqual(newChildProps, lastChildProps.current)) {
      lastChildProps.current = newChildProps;
      forceComponentUpdateDispatch();
    }
  });
  
  // ... 后面省略n行代码 ...
}
```
connect这块代码主要对应的是源码中`connectAdvanced`这个类，基本原理和结构跟我们这个都是一样的，只是他写的更灵活，支持用户传入自定义的`childPropsSelector`和合并`stateProps`, `dispatchProps`, `wrapperProps`的方法。

## 保证组件更新顺序
前面我们的Counter组件使用`connect`连接了redux store，假如他下面还有个子组件也连接到了redux store，我们就要考虑他们的回调的执行顺序的问题了。
我们知道React是单向数据流的，参数都是由父组件传给子组件的，现在引入了Redux，即使父组件和子组件都引用了同一个变量count，++但是子组件完全可以不从父组件拿这个参数，而是直接从Redux拿++，这样就打破了React本来的数据流向。++在父->子这种单向数据流中，如果他们的一个公用变量变化了，肯定是父组件先更新，然后参数传给子组件再更新++，但是在Redux里，数据变成了Redux -> 父，Redux -> 子，++父与子完全可以根据Redux的数据进行独立更新，而不能完全保证父级先更新，子级再更新的流程++。所以React-Redux花了不少功夫来手动保证这个更新顺序，React-Redux保证这个更新顺序的方案是在redux store外，再单独创建一个监听者类Subscription：

> 1. Subscription负责处理所有的state变化的回调
> 1. 如果当前连接redux的组件是**第一个**连接redux的组件，也就是说他是连接redux的根组件，他的state回调**直接注册到redux store**；同时++新建一个Subscription实例subscription通过context传递给子级++。
> 1. 如果当前连接redux的组件不是连接redux的根组件，也就是说他上面有组件已经注册到redux store了，那么他可以拿到上面通过context传下来的subscription，源码里面这个变量叫parentSub，++那当前组件的更新回调就注册到parentSub上++。同时**再新建一个Subscription实例**，**替代context上的subscription，继续往下传**，也就是说++他的**子组件**的回调会注册到当前subscription上++。
> 1. 当state变化了，++根组件++注册到redux store上的回调会++执行更新根组件++，同时++根组件需要手动执行子组件的回调++，子组件回调执行会++触发子组件更新++，然后++子组件再执行自己subscription上注册的回调++，++触发孙子组件更新++，孙子组件再调用注册到自己subscription上的回调。。。这样就实现了从根组件开始，一层一层更新子组件的目的，保证了父->子这样的更新顺序。

### Subscription类
所以我们先新建一个Subscription类：
```js
export default class Subscription {
  constructor(store, parentSub) {
    this.store = store
    this.parentSub = parentSub
    this.listeners = [];        // 源码listeners是用链表实现的，我这里简单处理，直接数组了

    this.handleChangeWrapper = this.handleChangeWrapper.bind(this)
  }

  // 子组件注册回调到Subscription上
  addNestedSub(listener) {
    this.listeners.push(listener)
  }

  // 执行子组件的回调
  notifyNestedSubs() {
    const length = this.listeners.length;
    for(let i = 0; i < length; i++) {
      const callback = this.listeners[i];
      callback();
    }
  }

  // 回调函数的包装
  handleChangeWrapper() {
    if (this.onStateChange) {
      this.onStateChange()
    }
  }

  // 注册回调的函数
  // 如果parentSub有值，就将回调注册到parentSub上
  // 如果parentSub没值，那当前组件就是根组件，回调注册到redux store上
  trySubscribe() {
      this.parentSub
        ? this.parentSub.addNestedSub(this.handleChangeWrapper)
        : this.store.subscribe(this.handleChangeWrapper)
  }
}
```

### 改造Provider
然后在我们前面自己实现的React-Redux里面，我们的根组件始终是Provider，所以Provider需要实例化一个Subscription并放到context上，而且每次state更新的时候需要手动调用子组件回调，代码改造如下：

```jsx
import React, { useMemo, useEffect } from 'react';
import ReactReduxContext from './Context';
import Subscription from './Subscription';

function Provider(props) {
  const {store, children} = props;

  // 这是要传递的context
  // 里面放入store和subscription实例
  const contextValue = useMemo(() => {
    const subscription = new Subscription(store)
    // 注册回调为通知子组件，这样就可以开始层级通知了
    subscription.onStateChange = subscription.notifyNestedSubs
    return {
      store,
      subscription
    }
  }, [store])

  // 拿到之前的state值
  const previousState = useMemo(() => store.getState(), [store])

  // 每次contextValue或者previousState变化的时候
  // 用notifyNestedSubs通知子组件
  useEffect(() => {
    const { subscription } = contextValue;
    subscription.trySubscribe()

    if (previousState !== store.getState()) {
      subscription.notifyNestedSubs()
    }
  }, [contextValue, previousState])

  // 返回ReactReduxContext包裹的组件，传入contextValue
  // 里面的内容就直接是children，我们不动他
  return (
    <ReactReduxContext.Provider value={contextValue}>
      {children}
    </ReactReduxContext.Provider>
  )
}

export default Provider; 
```

### 改造connect
有了Subscription类，connect就**不能直接注册到store**了，而是应该**注册到父级subscription上**，更新的时候**除了更新自己还要通知子组件更新**。在渲染包裹的组件时，也不能直接渲染了，而是应该++再次使用Context.Provider包裹下，传入修改过的contextValue++，这个contextValue里面的++subscription应该替换为自己的++。改造后代码如下：

```jsx
import React, { useContext, useRef, useLayoutEffect, useReducer } from 'react';
import ReactReduxContext from './Context';
import shallowEqual from './shallowEqual';
import Subscription from './Subscription';

function storeStateUpdatesReducer(count) {
  return count + 1;
}

function connect(
  mapStateToProps = () => {}, 
  mapDispatchToProps = () => {}
  ) {
  function childPropsSelector(store, wrapperProps) {
    const state = store.getState();   // 拿到state

    // 执行mapStateToProps和mapDispatchToProps
    const stateProps = mapStateToProps(state);
    const dispatchProps = mapDispatchToProps(store.dispatch);

    return Object.assign({}, stateProps, dispatchProps, wrapperProps);
  }

  return function connectHOC(WrappedComponent) {
    function ConnectFunction(props) {
      const { ...wrapperProps } = props;

      const contextValue = useContext(ReactReduxContext);

      const { store, subscription: parentSub } = contextValue;  // 解构出store和parentSub
      
      const actualChildProps = childPropsSelector(store, wrapperProps);

      const lastChildProps = useRef();
      useLayoutEffect(() => {
        lastChildProps.current = actualChildProps;
      }, [actualChildProps]);

      const [
        ,
        forceComponentUpdateDispatch
      ] = useReducer(storeStateUpdatesReducer, 0)

      // 新建一个subscription实例
      const subscription = new Subscription(store, parentSub);

      // state回调抽出来成为一个方法
      const checkForUpdates = () => {
        const newChildProps = childPropsSelector(store, wrapperProps);
        // 如果参数变了，记录新的值到lastChildProps上
        // 并且强制更新当前组件
        if(!shallowEqual(newChildProps, lastChildProps.current)) {
          lastChildProps.current = newChildProps;

          // 需要一个API来强制更新当前组件
          forceComponentUpdateDispatch();

          // 然后通知子级更新
          subscription.notifyNestedSubs();
        }
      };

      // 使用subscription注册回调
      subscription.onStateChange = checkForUpdates;
      subscription.trySubscribe();

      // 修改传给子级的context
      // 将subscription替换为自己的
      const overriddenContextValue = {
        ...contextValue,
        subscription
      }

      // 渲染WrappedComponent
      // 再次使用ReactReduxContext包裹，传入修改过的context
      return (
        <ReactReduxContext.Provider value={overriddenContextValue}>
          <WrappedComponent {...actualChildProps} />
        </ReactReduxContext.Provider>
      )
    }

    return ConnectFunction;
  }
}

export default connect;
```

## 参考
https://segmentfault.com/a/1190000023142285