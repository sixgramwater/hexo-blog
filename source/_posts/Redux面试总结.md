---
title: Redux面试总结
date: {{ date }}
tags: redux
cover: https://ppoffice.github.io/hexo-theme-icarus/gallery/covers/vector_landscape_1.svg
excerpt: 这篇文章主要针对一些面试中常见redux问题进行了解析，redux优化策略，参考了react-redux核心contributor的文章中的理解。此外对于redux中间件的原理进行了深入的分析。
toc: true
---

## Redux



### Redux解决了什么问题？为什么不使用Context?

https://blog.isquaredsoftware.com/2021/01/context-redux-differences/#what-is-react-context

#### 理解Context

Let's start by looking at [the actual description of Context from the React docs](https://reactjs.org/docs/context.html):

> **Context provides a way to pass data through the component tree without having to pass props down manually at every level**.
>
> In a typical React application, data is passed top-down (parent to child) via props, but this can be cumbersome for certain types of props (e.g. locale preference, UI theme) that are required by many components within an application. **Context provides a way to share values like these between components without having to explicitly pass a prop through every level of the tree**.

Notice that **it does not say anything about "managing" values - it only refers to "passing" and "sharing" values**.

需要注意的是：Context只是提供了一种“传递“、”分享”数据的方式，不需要再每个层级都手动的传递props。但是它跟“管理”数据无关。

#### 使用Context

需要注意：每当ParentComponent发生re-render, contextProvider中的value将会传递新的引用，这使得任何读取该context的组件将被强制re-render

```jsx
function ParentComponent() {
  const [counter, setCounter] = useState(0);

  // Create an object containing both the value and the setter
  const contextValue = {counter, setCounter};

  return (
    <MyContext.Provider value={contextValue}>
      <SomeChildComponent />
    </MyContext.Provider>
  )
}
```

A child component then can call `useContext` and read the value:

```jsx
function NestedChildComponent() {
  const { counter, setCounter } = useContext(MyContext);

  // do something with the counter value and setter
}
```

#### Context的目标与用途

基于上面讨论的内容，我们可以看到Context并没有“管理”任何东西。相反，它更像是一种“管道”或者“虫洞”。你使用 `<MyContext.Provider>`将某些东西放在管道的一端, 然后另一端的组件通过 `useContext(MyProvider)`来取得这些东西。

所以，**使用Context最主要的目的是避免“props drilling”**,为了传递一个值，我们不需要把他当作props进行一层层的手动传递。Context使得`<MyContext.Provider>`中的所有组件能够凭借`useContext(MyContext)`来按照需要获得某些东西。

概念上来讲，这是一种“依赖注入”。

#### 什么是Redux？

> **Redux is a pattern and library for managing and updating application state, using events called "actions".** It serves as a centralized store for state that needs to be used across your entire application, with rules ensuring that the state can only be updated in a predictable fashion.
>
> **Redux helps you manage "global" state** - state that is needed across many parts of your application.
>
> **The patterns and tools provided by Redux make it easier to understand when, where, why, and how the state in your application is being updated, and how your application logic will behave when those changes occur.**

Note that this description:

- specifically refers to "managing state"
- says that the purpose of Redux is to help you understand how state changes over time

#### React and Redux

Redux itself is UI-agnostic - you can use it with any UI layer (React, Vue, Angular, vanilla JS, etc), or without any UI at all.

That said, Redux is most commonly used with React. [The React-Redux library](https://react-redux.js.org/) is the official UI binding layer that lets React components interact with a Redux store by reading values from Redux state and dispatching actions. So, when most people refer to "Redux", they actually mean "using a Redux store and the React-Redux library together".

React-Redux allows any React component in the application to talk to the Redux store. This is only possible because **React-Redux uses Context internally**. However, it's critical to note that [**React-Redux only passes down the \*Redux store instance\* via context, not the current \*state value\*!**](https://blog.isquaredsoftware.com/2020/01/blogged-answers-react-redux-and-context-behavior/). This is actually an example of using Context for dependency injection, as mentioned above. We know that our Redux-connected React components need to talk to *a* Redux store, but we don't know or care *which* Redux store that is when we define the component. The actual Redux store is injected into the tree at runtime using the React-Redux `<Provider>` component.

Because of this, React-Redux can *also* be used to avoid prop-drilling, specifically because React-Redux uses Context internally. Instead of explicitly putting a new value into a `<MyContext.Provider>` yourself, you can put that data into the Redux store and then access it anywhere.

#### Purposes and Use Cases for (React-)Redux

The primary reason to use Redux is captured in the description from the Redux docs:

> **The patterns and tools provided by Redux make it easier to understand when, where, why, and how the state in your application is being updated, and how your application logic will behave when those changes occur.**

There are additional reasons why you might want to use Redux. "Avoiding prop-drilling" *is* one of those other reasons. Many people chose Redux early on specifically to let them avoid prop-drilling, because React's legacy context was broken and React-Redux worked correctly.

Other valid reasons to use Redux include:

- Wanting to write your state management logic completely separate from the UI layer
- Sharing state management logic between different UI layers (such as an application that is being migrated from AngularJS to React)
- Using the power of Redux middleware to add additional logic when actions are dispatched
- Being able to persist portions of the Redux state
- Enabling bug reports that can be replayed by developers
- Faster debugging of logic and UI while in development

Dan Abramov listed a number of these use cases when he wrote his post [You Might Not Need Redux](https://medium.com/@dan_abramov/you-might-not-need-redux-be46360cf367), all the way back in 2016.

#### 为什么Context不是一个状态管理工具

## Redux实现原理

### createStore

`createStore`：这个API接受reducer方法作为参数，返回一个store，主要功能都在这个store上。

看看store上我们都用到了啥：

- `store.subscribe`: 订阅state的变化，当state变化的时候执行回调，可以有多个subscribe，里面的回调会依次执行。
- `store.dispatch`: 发出action的方法，每次dispatch action都会执行reducer生成新的state，然后执行subscribe注册的回调。
- `store.getState`:一个简单的方法，返回当前的state。

`subscribe`注册回调，`dispatch`触发回调，因此这是一个典型的**发布订阅模式**。

```js
function createStore(reducer) {
    var state;
    var listeners = []

    function getState() {
        return state
    }
    
    function subscribe(listener) {
        listeners.push(listener)
        return function unsubscribe() {
            var index = listeners.indexOf(listener)
            listeners.splice(index, 1)
        }
    }
    
    function dispatch(action) {
        state = reducer(state, action)
        listeners.forEach(listener => listener())
    }

    dispatch({})

    return { dispatch, subscribe, getState }
}
```

### combineReducers

要手写`combineReducers`，我们先来分析下他干了啥，首先它的返回值是一个reducer，这个reducer同样会作为`createStore`的参数传进去，说明这个返回值是一个跟我们之前普通reducer结构一样的函数。这个函数同样接收state和action然后返回新的state，只是这个新的state要符合`combineReducers`参数的数据结构。我们尝试来写下： 

```js
function combineReducers(reducerMap) {
  const reducerKeys = Object.keys(reducerMap);    // 先把参数里面所有的键值拿出来
  
  // 事实上是遍历reducer，每次都会调用所有reducer，并生成相应的state
  const reducer = (state = {}, action) => {
    const newState = {};
    // 遍历传进去的reducerMap，获得key和相对应的reducer
    for(let i = 0; i < reducerKeys.length; i++) {
      // reducerMap里面每个键的值都是一个reducer，我们把它拿出来运行下就可以得到对应键新的state值
      // 然后将所有reducer返回的state按照参数里面的key组装好
      // 最后再返回组装好的newState就行
      const key = reducerKeys[i];
      const currentReducer = reducerMap[key];
      const prevState = state[key];
      newState[key] = currentReducer(prevState, action);
    }
    // state通过key分割成多个
    return newState;
  };
  
  return reducer;
}
```

### Redux中间件原理

#### Middleware作用的位置

Redux 中间件解决的问题与 Express 或 Koa 中间件不同，但在概念上是相似的。**<u>它在 dispatch action 和到达 reducer 的那一刻之间提供了逻辑插入点</u>**。可以使用 Redux 中间件进行日志记录、异常监控、与异步 API 对话、路由等。

那么中间件在redux中作用于什么位置呢？

<img src="https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/9/10/165c24bf96cbce24~tplv-t2oaga2asx-zoom-in-crop-mark:3024:0:0:0.awebp" alt="redux工作流程_中间件" style="zoom:80%;" />

1. `dispatch`一个`action`（不一定是标准的action）
2. 这个`action`先被`middleware`处理（比如在这里发送一个异步请求）
3. 中间件处理结束后，再发送一个`action`（有可能是原来的`action`，也可能是不同的`action`,因中间件功能不同而不同）
4. 中间件发出的`action`可能继续被另一个中间件处理，进行类似3的步骤。即中间件可以**链式串联**。
5. 最后一个中间件处理完后，`dispatch`一个符合`reducer`处理标准的`action`（纯对象action）
6. 这个标准的`action`被`reducer`处理，
7. `reducer`根据action更新`state`



#### Middleware的写法

当我想写一个logger的中间件时，我可以这样写
```js
// logger是一个中间件，注意返回值嵌了好几层函数
// 我们后面来看看为什么这么设计
function logger(store) {
  return function(next) {
    return function(action) {
      console.group(action.type);
      console.info('dispatching', action);
      let result = next(action);
      console.log('next state', store.getState());
      console.groupEnd();
      return result
    }
  }
}

// 在createStore的时候将applyMiddleware作为第二个参数传进去
const store = createStore(
  reducer,
  applyMiddleware(logger)
)
```

可以看到上述代码为了支持中间件，`createStore`支持了第二个参数，这个参数官方称为`enhancer`，顾名思义他是一个增强器，用来增强store的能力的。官方对于enhancer的定义如下：
```typescript
type StoreEnhancer = (next: StoreCreator) => StoreCreator
```

这个`enhancer`函数接受一个「普通createStore函数」作为参数，返回一个「加强后的createStore函数」。

这个加强的过程中做的事情，其实就是改造dispatch，添加上中间件。

`applyMiddleware`方法的返回值就是一个`enhancer`.

#### Redux中间件的构成

```js
// Middleware written as ES5 functions

// Outer function:
function exampleMiddleware(storeAPI) {
  return function wrapDispatch(next) {
    return function handleAction(action) {
      // Do anything here: pass the action onwards with next(action),
      // or restart the pipeline with storeAPI.dispatch(action)
      // Can also use storeAPI.getState() here

      return next(action)
    }
  }
}
```

让我们分解一下这三个部分分别起了什么作用：

- `exampleMiddleware`: 最外层的函数就是所谓的middleware. 它会被 `applyMiddleware`所调用, 并且接收一个 `storeAPI` 对象，该对象包含了store的 `{dispatch, getState}` 方法. 中间件函数的输入是store的`getState`和`dispatch`，输出为wrapDispatch函数（改造、包裹`dispatch`的函数）
- `wrapDispatch`: 输入是一个`dispatch`，输出「改造后的`dispatch`」
- `handleAction`：最终，内层函数接收当前的`action`作为参数，每当dispatch 一个action时都会被调用。

> - `exampleMiddleware`: The outer function is actually the "middleware" itself. It will be called by `applyMiddleware`, and receives a `storeAPI` object containing the store's `{dispatch, getState}` functions. These are the same `dispatch` and `getState` functions that are actually part of the store. If you call this `dispatch` function, it will send the action to the *start* of the middleware pipeline. This is only called once.
> - `wrapDispatch`: The middle function receives a function called `next` as its argument. This function is actually the *next middleware* in the pipeline. If this middleware is the last one in the sequence, then `next` is actually the original `store.dispatch` function instead. Calling `next(action)` passes the middleware to the *next* middleware in the pipeline. This is also only called once
> - `handleAction`: Finally, the inner function receives the current `action` as its argument, and will be called *every* time an action is dispatched.

#### createStore支持enhancer

```js
function createStore(reducer, enhancer) {   // 接收第二个参数enhancer
  // 先处理enhancer
  // 如果enhancer存在并且是函数
  // 我们将createStore作为参数传给他
  // 他应该返回一个新的createStore给我
  // 我再拿这个新的createStore执行，应该得到一个store
  // 直接返回这个store就行
  if(enhancer && typeof enhancer === 'function'){
    const newCreateStore = enhancer(createStore);
    const newStore = newCreateStore(reducer);
    return newStore;
  }
  
  // 如果没有enhancer或者enhancer不是函数，直接执行之前的逻辑
  // 下面这些代码都是之前那版
  // 省略n行代码
    // .......
  const store = {
    subscribe,
    dispatch,
    getState
  }

  return store;
}
```

#### applyMiddleware返回值是一个enhancer
前面我们已经有了enhancer的基本结构，`applyMiddleware`是作为第二个参数传给`createStore`的，也就是说他是一个enhancer，准确的说是`applyMiddleware`的返回值是一个enhancer，因为我们传给`createStore`的是他的执行结果`applyMiddleware()`：

```js
function applyMiddleware(middleware) {
  // applyMiddleware的返回值应该是一个enhancer
  // 按照我们前面说的enhancer的参数是createStore
  function enhancer(createStore) {
    // enhancer应该返回一个新的createStore
    function newCreateStore(reducer) {
      // 我们先写个空的newCreateStore，直接返回createStore的结果
      const store = createStore(reducer);
      return store
    }
    
    return newCreateStore;
  }
  
  return enhancer;
}
```

#### 实现applyMiddleware
一个中间件到底有什么功能，还是以前面的logger中间件为例：
```js
function logger(store) {
    // 这里的next就是原始的dispatch函数
    // 也就是说middleware(store)调用后的函数为: (dispatch) => newDispatch
  return function(next) {
    // return的其实是一个全新的、具有额外功能的dispatch(action)函数
    return function(action) {
      console.group(action.type);
      console.info('dispatching', action);
      let result = next(action);  // 就是原始的dispatch(action)
      console.log('next state', store.getState());
      console.groupEnd();
      return result
    }
  }
}
```
这里的`next(action)`就是`dispatch(action)`，
最后一层返回值`return function(action)`的结构，
其实他就是一个新的dispatch(action)。
这个新的dispatch(action)会调用原始的dispatch，并且在调用的前后加上自己的逻辑。

所以我们可以梳理出中间件的结构：
1. 一个中间件接收store作为参数，会返回一个函数
1. 返回的这个函数接收老的dispatch函数作为参数，会返回一个新的函数
1. 返回的新函数就是新的dispatch函数，这个函数里面可以拿到外面两层传进来的store和老dispatch函数(action)

所以其实是一个`装饰器模式`

`applyMiddleware`接受middleware为参数，并返回一个enhancer

```js
// 直接把前面的结构拿过来
function applyMiddleware(middleware) {
  function enhancer(createStore) {
    function newCreateStore(reducer) {
      const store = createStore(reducer);
      
      // 将middleware拿过来执行下，传入store
      // 得到第一层函数
      const func = middleware(store);
      
      // 解构出原始的dispatch
      const { dispatch } = store;
      
      // 将原始的dispatch函数传给func执行
      // 得到增强版的dispatch
      const newDispatch = func(dispatch);
      
      // 返回的时候用增强版的newDispatch替换原始的dispatch
      return {...store, dispatch: newDispatch}
    }
    
    return newCreateStore;
  }
  
  return enhancer;
}
```

所以`enhancer(createStore) => createStore`函数就是将原始的`createStore`转化为一个返回newDispatch函数的newCreateStore(可以理解为enhancedCreateStore)。
`applyMiddleware`函数的主要作用就是返回这个enhancer.

#### 支持多个middleware
我们的`applyMiddleware`还差一个功能，就是支持多个middleware，比如像这样:
```js
applyMiddleware(
  rafScheduler,
  timeoutScheduler,
  thunk,
  vanillaPromise,
  readyStatePromise,
  logger,
  crashReporter
)
```

其实要支持这个也简单，我们返回的newDispatch里面依次的将传入的middleware拿出来执行就行，多个函数的串行执行可以使用辅助函数`compose`，这个函数定义如下。只是需要注意的是我们这里的`compose`不能把方法拿来执行就完了，应该返回一个包裹了所有方法的方法。
```js
function compose(...func){
  return funcs.reduce((a, b) => (...args) => a(b(...args)));
}
```

支持多个middleware的`applyMiddleware`
```js
// 参数支持多个中间件
function applyMiddleware(...middlewares) {
  function enhancer(createStore) {
    function newCreateStore(reducer) {
      const store = createStore(reducer);
      
      // 多个middleware，先解构出dispatch => newDispatch的结构
      const chain = middlewares.map(middleware => middleware(store));
      const { dispatch } = store;
      
      // 用compose得到一个组合了所有newDispatch的函数
      const newDispatchGen = compose(...chain);
      // 执行这个函数得到newDispatch
      const newDispatch = newDispatchGen(dispatch);

      return {...store, dispatch: newDispatch}
    }
    
    return newCreateStore;
  }
  
  return enhancer;
}
```
现在我们也可以知道他的中间件为什么要包裹几层函数了：

第一层：目的是传入store参数

第二层：第二层的结构是dispatch => newDispatch，多个中间件的这层函数可以compose起来，形成一个大的dispatch => newDispatch

第三层：这层就是最终的返回值了，其实就是newDispatch，是增强过的dispatch，是中间件的真正逻辑所在。

### Redux中间件洋葱模型

https://juejin.cn/post/6844903597776306190

关键词：洋葱模型、闭包、柯里化

## React Redux实现原理

Using Redux with *any* UI layer requires [the same consistent set of steps](https://blog.isquaredsoftware.com/presentations/workshops/redux-fundamentals/ui-layer.html#/4):

1. Create a Redux store
2. Subscribe to updates
3. Inside the subscription callback:
   1. Get the current store state
   2. Extract the data needed by this piece of UI
   3. Update the UI with the data
4. If necessary, render the UI with initial state
5. Respond to UI inputs by dispatching Redux actions

### connect

这里是Dan Abramov所写的最小版本，是connect的概念性实现。

```js
// connect() is a function that injects Redux-related props into your component.
// You can inject data and callbacks that change that data by dispatching actions.
function connect(mapStateToProps, mapDispatchToProps) {
  // It lets us inject component as the last step so people can use it as a decorator.
  // Generally you don't need to worry about it.
  return function (WrappedComponent) {
    // It returns a component
    return class extends React.Component {
      render() {
        return (
          // that renders your component
          <WrappedComponent
            {/* with its props  */}
            {...this.props}
            {/* and additional props calculated from Redux store */}
            {...mapStateToProps(store.getState(), this.props)}
            {...mapDispatchToProps(store.dispatch, this.props)}
          />
        )
      }
      
      componentDidMount() {
        // it remembers to subscribe to the store so it doesn't miss updates
        this.unsubscribe = store.subscribe(this.handleChange.bind(this))
      }
      
      componentWillUnmount() {
        // and unsubscribe later
        this.unsubscribe()
      }
    
      handleChange() {
        // and whenever the store state changes, it re-renders.
        this.forceUpdate()
      }
    }
  }
}

// This is not the real implementation but a mental model.
// It skips the question of where we get the "store" from (answer: `<Provider>` puts it in React context)
// and it skips any performance optimizations (real connect() makes sure we don't re-render in vain).

// The purpose of connect() is that you don't have to think about
// subscribing to the store or perf optimizations yourself, and
// instead you can specify how to get props based on Redux store state
```

可以看到:

- connect是一个HOC，它返回一个WrappedComponent
- The wrapped component's props are a combination of the wrapper component's props, the values from `mapState`, and the values from `mapDispatch`.
- Each wrapper component is an individual subscriber to the Redux store.
- The wrapper component abstracts away the details of *which* store you're using, *how* it's interacting with the store, and optimizing performance so that your own component only re-renders when it needs to.
- You simply specify how to extract the data your component needs based on the store state, and the functions it can call to dispatch actions to the store

但是，这个实现还有一些细节没有考虑：

- store从哪里获得？
- `connect`如何知道组件确实**需要更新**
- 如何优化？

## Redux优化

https://github.com/shaozj/blog/issues/36

### 重复渲染优化

场景：10k长度的列表元素需要渲染，点击其中一个元素，使其高亮。

问题：如果不采取任何手段，每次点击都会引发10k条元素全都重新渲染

https://zhuanlan.zhihu.com/p/32601923

解决方案：

- 优化数据结构：使得要渲染的数据结构保持不变，另外单独维护一个元素高亮状态的结构
- 避免其他元素的re-render：shouldComponentUpdate, PureComponent, Memo等手段
- 使用虚拟化、窗口化等技术。



## React-redux版本迭代（v5,v6,v7)

https://github.com/reduxjs/react-redux/issues/1177

### React-redux v6

每当store变化，都会导致根组件re-render，但是根组件的子组件并不应该re-render,这用到了我们上文提到的props.children来解决这个问题。

在react-redux v7中，其实并不存在这个问题，store

> Internal Changes Direct Component Subscriptions
>
> In v6, we switched from individual components subscribing to the store, to having `<Provider>` subscribe and components read the store state from React's Context API. This worked, but unfortunately the Context API isn't as optimized for frequent updates as we'd hoped, and our usage patterns led to some folks reporting performance issues in some scenarios.
>
> In v7, we've switched back to using direct subscriptions internally, which should improve performance considerably.
>
> (This does result in some changes that are visible to user-facing code, in that updates dispatched in React lifecycle methods are immediately reflected in later component updates. Examples of this include components dispatching while mounting in an SSR environment. This was the behavior through v5, and is not considered part of our public API.)

## 记忆化 Selectors[#](http://cn.redux.js.org/tutorials/fundamentals/part-7-standard-patterns#记忆化-selectors)

http://cn.redux.js.org/tutorials/fundamentals/part-7-standard-patterns#%E8%AE%B0%E5%BF%86%E5%8C%96-selectors

我们已经看到我们可以编写 "selector" 函数，它接受 Redux `state` 对象作为参数，并返回一个值：

const selectTodos = state => state.todos

Copy

假如我们需要 *派生* 一些数据怎么办？举个例子，或许我们希望只要一个 todo IDs 组成的数组：

const selectTodoIds = state => state.todos.map(todo => todo.id)

Copy

然而，`array.map()` 每次都返回的是一个新数组引用（reference）。我们知道 *每次* dispatch action 后 React-Redux `useSelector` hook 都会重新调用其 selector 函数，如果 selector 返回一个新值，组件一定会重新渲染。

在这个例子中，**在 \*每个\* action 后调用 `useSelector(selectTodoIds)` 将 \*总是\* 造成重渲染，因为总是返回一个新数组引用`**

在第 5 节，我们看到[我们传了 `shallowEqual` 作为参数给 `useSelector`](http://cn.redux.js.org/tutorials/fundamentals/part-5-ui-react#selecting-data-in-list-items-by-id)。现在有另一选择：我们可以使用 “记忆化的（memoized）” selectors。

**记忆化** 是缓存技术的一种 - 具体来说，保存昂贵计算的结果，如果我们以后看到相同的输入，请重用这些结果。

**记忆化 selector functions** 是保存最新结果值的 selector，如果使用相同的输入多次调用它们，则将返回相同的结果值。如果使用与上次不同的 *different* 输入调用它们，它们将重新计算新的结果值，缓存该值，然后返回新结果。

### 结合 `createSelector` 记忆化 Selectors[#](http://cn.redux.js.org/tutorials/fundamentals/part-7-standard-patterns#结合-createselector-记忆化-selectors)

**[Reselect library](https://github.com/reduxjs/reselect) 提供了一个能生成记忆化 selector 函数的 `createSelector` API**。该 API 接收一个或多个 "input selector" 函数作为参数，加上一个 "output selector"，并返回新的 selector 函数。每次调用选择器时：

- 所有 "input selectors" 都使用所有参数调用
- 如果任何 input selector 返回值已更改，"output selector" 将重新运行
- 所有 input selector 的结果都将成为 output selector 的参数
- output selector 的最终结果将缓存以供下次使用

让我们来创建一个记忆化版的 `selectTodoIds`，并且在 `<TodoList>` 中使用。

首先我们需要安装 Reselect:

npm install reselect

Copy

接着我们导入且调用 `createSelector`。我们最初的 `selectTodoIds` 函数是在 `TodoList.js` 中定义的，但更常见的是选择器函数写在相关的切片文件中。因此，让我们将其添加到待办事项切片中：

src/features/todos/todosSlice.js

import { createSelector } from 'reselect'



// omit reducer



// omit action creators



export const selectTodoIds = createSelector(

  // First, pass one or more "input selector" functions:

  state => state.todos,

  // Then, an "output selector" that receives all the input results as arguments

  // and returns a final result value

  todos => todos.map(todo => todo.id)

)

Copy

接下来, 在 `<TodoList>` 里使用:

src/features/todos/TodoList.js

import React from 'react'

import { useSelector, shallowEqual } from 'react-redux'



import { selectTodoIds } from './todosSlice'

import TodoListItem from './TodoListItem'



const TodoList = () => {

  const todoIds = useSelector(selectTodoIds)



  const renderedListItems = todoIds.map(todoId => {

​    return <TodoListItem key={todoId} id={todoId} />

  })



  return <ul className="todo-list">{renderedListItems}</ul>

}

Copy

这实际上与 `shallowEqual` 比较函数的行为略有不同。每当 `state.todos` 数组更改时，我们都会创建一个新的 todo IDs 数组。这包括对待办事项的任何不可变更新，例如切换其 `completed` 字段，因为我们必须为不可变更新创建一个新数组。

##### TIP

仅当您实际从原始数据派生其他值时，记忆选择器才有用。如果只是查找并返回现有值，则可以将选择器保留为普通函数。

### 具有多个参数的 Selectors[#](http://cn.redux.js.org/tutorials/fundamentals/part-7-standard-patterns#具有多个参数的-selectors)

我们的待办事项应用程序应该能够根据其完成状态过滤可见的待办事项。让我们编写一个记忆选择器，返回经过过滤的待办事项列表。

我们知道我们需要整个 `todos` 数组作为 output selector 的一个参数。我们还需要传入当前完成状态筛选器值。我们将添加一个单独的 "input selector" 来提取每个值，并将结果传递给 "output selector"。

src/features/todos/todosSlice.js

import { createSelector } from 'reselect'

import { StatusFilters } from '../filters/filtersSlice'



// omit other code



export const selectFilteredTodos = createSelector(

  // First input selector: all todos

  state => state.todos,

  // Second input selector: current status filter

  state => state.filters.status,

  // Output selector: receives both values

  (todos, status) => {

​    if (status === StatusFilters.All) {

​      return todos

​    }



​    const completedStatus = status === StatusFilters.Completed

​    // Return either active or completed todos based on filter

​    return todos.filter(todo => todo.completed === completedStatus)

  }

)

Copy

:::警告

请注意，我们现在在两个切片之间添加了一个导入依赖关系 - `todosSlice` 正在从 `filtersSlice` 导入一个值。这是合法的，但要小心。**如果两个切片 \*都在\* 尝试从彼此导入某些内容，则最终可能会遇到“循环导入依赖项”问题，从而导致代码崩溃**。如果发生这种情况，请尝试将一些常用代码移动到其自己的文件中，然后改为从该文件导入。

:::

现在，我们可以使用这个新的 "filtered todos" 选择器作为另一个选择器的输入，该选择器返回这些待办事项的 ID：

src/features/todos/todosSlice.js

export const selectFilteredTodoIds = createSelector(

  // Pass our other memoized selector as an input

  selectFilteredTodos,

  // And derive data in the output selector

  filteredTodos => filteredTodos.map(todo => todo.id)

)

Copy

如果我们在 `<TodoList>` 使用 `selectFilteredTodoIds`，那么我们应该能够将几个待办事项标记为已完成：

![Todo app - todos marked completed](http://cn.redux.js.org/assets/images/todos-app-markedCompleted-fe1ddaba1c85e3f9841728d39459d126.png)

然后将列表筛选为 *只* 显示已完成的待办事项：

![Todo app - todos marked completed](http://cn.redux.js.org/assets/images/todos-app-showCompleted-6eeff512e106457fd4fb469d2f66337d.png)

然后，我们可以扩展我们的 `selectFilteredTodos`，以便在选择中也包括颜色过滤：

src/features/todos/todosSlice.js

export const selectFilteredTodos = createSelector(

  // First input selector: all todos

  selectTodos,

  // Second input selector: all filter values

  state => state.filters,

  // Output selector: receives both values

  (todos, filters) => {

​    const { status, colors } = filters

​    const showAllCompletions = status === StatusFilters.All

​    if (showAllCompletions && colors.length === 0) {

​      return todos

​    }



​    const completedStatus = status === StatusFilters.Completed

​    // Return either active or completed todos based on filter

​    return todos.filter(todo => {

​      const statusMatches =

​        showAllCompletions || todo.completed === completedStatus

​      const colorMatches = colors.length === 0 || colors.includes(todo.color)

​      return statusMatches && colorMatches

​    })

  }

)

Copy

请注意，通过将逻辑封装在此选择器中，即使更改了筛选行为，我们的组件也不需要更改。现在我们可以同时按状态和颜色进行过滤：

![Todo app - status and color filters](http://cn.redux.js.org/assets/images/todos-app-selectorFilters-9ebfc37b389c997e011aff74d525a1aa.png)

最后，我们的代码在几个地方查找 `state.todos`。在完成本节的其余部分时，我们将对该状态的设计方式进行一些更改，因此我们将提取一个 `selectTodos` 选择器并在任何地方使用它。我们还可以将 `selectTodoById` 移动到 `todosSlice` 中：

src/features/todos/todosSlice.js

export const selectTodos = state => state.todos



export const selectTodoById = (state, todoId) => {

  return selectTodos(state).find(todo => todo.id === todoId)

}

Copy

##### INFO

学习更多关于怎样使用 Reselect 和记忆化 selectors ，请看：

- The [Reselect docs](https://github.com/reduxjs/reselect)
- [Idiomatic Redux: Using Reselect Selectors for Encapsulation and Performance](https://blog.isquaredsoftware.com/2017/12/idiomatic-redux-using-reselect-selectors/)
