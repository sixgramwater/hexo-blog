---
title: Redux优化
date: {{ date }}
tags: redux
cover: https://ppoffice.github.io/hexo-theme-icarus/gallery/covers/vector_landscape_3.svg
excerpt: 参考了react-redux核心contributor的文章中的理解redux优化策略,参考了react-redux核心contributor的文章中的理解。
toc: true
---
## mapStateToProps Guide
### Let `mapStateToProps` Reshape the Data from the Store
mapStateToProps可以、也应当不只是返回`return state.someSlice`这样的工作。
> They have the responsibility of "re-shaping" store data as needed for that component. This may include returning a value as a specific prop name, combining pieces of data from different parts of the state tree, and transforming the store data in different ways.

## Selectors 优化
### Deriving Data
We specifically recommend that Redux apps should keep the Redux state minimal, and derive additional values from that state whenever possible.

This includes things like calculating filtered lists or summing up values. As an example, a todo app would keep an original list of todo objects in state, but derive a filtered list of todos outside the state whenever the state is updated. Similarly, a check for whether all todos have been completed, or number of todos remaining, can be calculated outside the store as well.

This has several benefits:

- The actual state is easier to read
- Less logic is needed to calculate those additional values and keep them in sync with the rest of the data
- The original state is still there as a reference and isn't being replaced

### Encapsulating State Shape with Selectors
考虑使用selector function进行封装
```js
const data = useSelector(state => state.some.deeply.nested.field)
```
That is legal code, and will run fine. But, it might not be the best idea architecturally. Imagine that you've got several components that need to access that field. What happens if you need to make a change to where that piece of state lives? You would now have to go change every useSelector hook that references that value. So, in the same way that we recommend using action creators to encapsulate details of creating actions, we recommend defining reusable selectors to encapsulate the knowledge of where a given piece of state lives. Then, you can use a given selector function many times in the codebase, anywhere that your app needs to retrieve that particular data.


### Optimizing Selectors with Memoization
Selector functions often need to perform relatively "expensive" calculations, or create derived values that are new object and array references. This can be a concern for application performance, for several reasons:

- Selectors used with `useSelector` or `mapState` will be **re-run after every dispatched action**, regardless of what section of the Redux root state was actually updated. Re-running expensive calculations when the input state sections didn't change is a waste of CPU time, and it's very likely that the inputs won't have changed most of the time anyway.
- useSelector and mapState rely on === **reference equality** checks of the return values to determine if the component needs to re-render. If a selector always returns new references, it will force the component to re-render even if the derived data is effectively the same as last time. This is especially common with array operations like map() and filter(), which return new array references.

As an example, this component is written badly, because its `useSelector` call **always returns a new array reference**. That means the component will **re-render after every dispatched action**, even if the input state.todos slice hasn't changed:

```js
function TodoList() {
  // ❌ WARNING: this _always_ returns a new reference, so it will _always_ re-render!
  const completedTodos = useSelector(state =>
    state.todos.map(todo => todo.completed)
  )
}
```
Another example is a component that needs to do some "expensive" work to transform data:
```js
function ExampleComplexComponent() {
  const data = useSelector(state => {
    const initialData = state.data
    const filteredData = expensiveFiltering(initialData)
    const sortedData = expensiveSorting(filteredData)
    const transformedData = expensiveTransformation(sortedData)

    return transformedData
  })
}
```
Similarly, this "expensive" logic will re-run after every dispatched action. Not only will it probably create new references, but it's work that doesn't need to be done unless `state.data` actually changes.

Because of this, we need a way to write optimized selectors that can avoid recalculating results if the same inputs are passed in. This is where the idea of memoization comes in.

总结：
我们常常会在selectors中获取derived data，但是这些操作有可能会有以下的问题：
1. 总是返回全新的引用(例如Array.map()总是返回一个全新的数组引用)，这会使得每当dispatch一个action时页面都会re-render
2. 在selector中进行昂贵的计算（计算衍生值、数据变换等），而这些计算每当dispatch action时，都会重复执行。

因此我们需要引入memoization selector来优化这一问题。

### Writing Memoized Selectors with Reselect
redux生态中通常使用reselect来创建memoized selector functions。
#### `createSelector`
```js
const selectA = state => state.a
const selectB = state => state.b
const selectC = state => state.c

const selectABC = createSelector([selectA, selectB, selectC], (a, b, c) => {
  // do something with a, b, and c, and return a result
  return a + b + c
})

// Call the selector function and get a result
const abc = selectABC(state)

// could also be written as separate arguments, and works exactly the same
const selectABC2 = createSelector(selectA, selectB, selectC, (a, b, c) => {
  // do something with a, b, and c, and return a result
  return a + b + c
})
```
When you call the selector, Reselect will run your input selectors with all of the arguments you gave, and looks at the returned values. If any of the results are === different than before, it will re-run the output selector, and pass in those results as the arguments. If all of the results are the same as the last time, it will skip re-running the output selector, and just return the cached final result from before.

This means that "input selectors" should usually **just extract and return values**, and the "output selector" should **do the transformation work**.