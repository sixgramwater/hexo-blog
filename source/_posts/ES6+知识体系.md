---
title: ES6知识体系总结
date: {{ date }}
tags: JS
cover: https://ppoffice.github.io/hexo-theme-icarus/gallery/covers/vector_landscape_1.svg
excerpt: A total review of ES6
toc: true
---
## Proxy & Reflect

## symbol类型

https://zh.javascript.info/symbol

### symbol

“symbol” 值表示唯一的标识符。

可以使用 `Symbol()` 来创建这种类型的值：

```javascript
let id = Symbol();
```

创建时，我们可以给 symbol 一个描述（也称为 symbol 名），这在代码调试时非常有用：

```javascript
// id 是描述为 "id" 的 symbol
let id = Symbol("id");
```

symbol 保证是唯一的。即使我们创建了许多具有相同描述的 symbol，它们的值也是不同。描述只是一个标签，不影响任何东西。

例如，这里有两个描述相同的 symbol —— 但是它们不相等：

```javascript
let id1 = Symbol("id");
let id2 = Symbol("id");

alert(id1 == id2); // false
```

### symbol作为对象属性

symbol 允许我们创建对象的“隐藏”属性，代码的任何其他部分都不能意外访问或重写这些属性。

```javascript
let user = { // 属于另一个代码
  name: "John"
};

let id = Symbol("id");

user[id] = 1;

alert( user[id] ); // 我们可以使用 symbol 作为键来访问数据
```

使用 `Symbol("id")` 作为键，比起用字符串 `"id"` 来有什么好处呢？

### 对象字面量中的symbol

如果我们要在对象字面量 `{...}` 中使用 symbol，则需要使用方括号把它括起来。

就像这样：

```javascript
let id = Symbol("id");

let user = {
  name: "John",
  [id]: 123 // 而不是 "id"：123
};
```

这是因为我们需要变量 `id` 的值作为键，而不是字符串 “id”。

### symbol 在 for…in 中会被跳过

symbol 属性不参与 `for..in` 循环。

例如：

```javascript
let id = Symbol("id");
let user = {
  name: "John",
  age: 30,
  [id]: 123
};

for (let key in user) alert(key); // name, age（没有 symbol）

// 使用 symbol 任务直接访问
alert( "Direct: " + user[id] );
```

[Object.keys(user)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/keys) 也会忽略它们。这是一般“隐藏符号属性”原则的一部分。如果另一个脚本或库遍历我们的对象，它不会意外地访问到符号属性。

相反，[Object.assign](https://developer.mozilla.org/zh/docs/Web/JavaScript/Reference/Global_Objects/Object/assign) 会同时复制字符串和 symbol 属性：

```javascript
let id = Symbol("id");
let user = {
  [id]: 123
};

let clone = Object.assign({}, user);

alert( clone[id] ); // 123
```

这里并不矛盾，就是这样设计的。这里的想法是当我们克隆或者合并一个 object 时，通常希望 **所有** 属性被复制（包括像 `id` 这样的 symbol）。

### 全局symbol

通常所有的 symbol 都是**不同**的，即使它们有相同的名字。正如在一开始使用`Symbol()`可以创建一个symbol的原始值，而且即使description不同，它们也是唯一的，永远不会相等。

但有时我们想要名字相同的 symbol 具有相同的实体。例如，应用程序的不同部分想要访问的 symbol `"id"` 指的是完全相同的属性。

为了实现这一点，这里有一个 **全局 symbol 注册表**。我们可以在其中创建 symbol 并在稍后访问它们，它可以确保每次访问相同名字的 symbol 时，返回的都是相同的 symbol。

要实现这一点，需要使用`Symbol.for(key)`

该调用会检查全局注册表，如果有一个描述为 `key` 的 symbol，则返回该 symbol，否则将创建一个新 symbol（`Symbol(key)`），并通过给定的 `key` 将其存储在注册表中。

例如：

```javascript
// 从全局注册表中读取
let id = Symbol.for("id"); // 如果该 symbol 不存在，则创建它

// 再次读取（可能是在代码中的另一个位置）
let idAgain = Symbol.for("id");

// 相同的 symbol
alert( id === idAgain ); // true
```

#### Symbol.keyFor

对于全局 symbol，不仅有 `Symbol.for(key)` 按名字返回一个 symbol，还有一个反向调用：`Symbol.keyFor(sym)`，它的作用完全反过来：通过全局 symbol 返回一个名字。

例如：

```javascript
// 通过 name 获取 symbol
let sym = Symbol.for("name");
let sym2 = Symbol.for("id");

// 通过 symbol 获取 name
alert( Symbol.keyFor(sym) ); // name
alert( Symbol.keyFor(sym2) ); // id
```

`Symbol.keyFor` 内部使用全局 symbol 注册表来查找 symbol 的键。所以它不适用于非全局 symbol。如果 symbol 不是全局的，它将无法找到它并返回 `undefined`。

也就是说，任何 symbol 都具有 `description` 属性。

例如：

```javascript
let globalSymbol = Symbol.for("name");
let localSymbol = Symbol("name");

alert( Symbol.keyFor(globalSymbol) ); // name，全局 symbol
alert( Symbol.keyFor(localSymbol) ); // undefined，非全局

alert( localSymbol.description ); // name
```

### 系统symbol

JavaScript 内部有很多“系统” symbol，我们可以使用它们来微调对象的各个方面。

它们都被列在了 [众所周知的 symbol](https://tc39.github.io/ecma262/#sec-well-known-symbols) 表的规范中：

- `Symbol.hasInstance`
- `Symbol.isConcatSpreadable`
- `Symbol.iterator`
- `Symbol.toPrimitive`
- ……等等。

### 总结

> - `symbol` 是唯一标识符的基本类型
>
> - symbol 是使用带有可选描述（name）的 `Symbol()` 调用创建的。
>
> - symbol 总是不同的值，即使它们有相同的名字。如果我们希望同名的 symbol 相等，那么我们应该使用全局注册表：`Symbol.for(key)` 返回（如果需要的话则创建）一个以 `key` 作为名字的全局 symbol。使用 `Symbol.for` 多次调用 `key` 相同的 symbol 时，返回的就是同一个 symbol。
>
> - symbol 有两个主要的使用场景：
>
> - 1. “隐藏” 对象属性。
>
>      如果我们想要向“属于”另一个脚本或者库的对象添加一个属性，我们可以创建一个 symbol 并使用它作为属性的键。symbol 属性不会出现在 `for..in` 中，因此它不会意外地被与其他属性一起处理。并且，它不会被直接访问，因为另一个脚本没有我们的 symbol。因此，该属性将受到保护，防止被意外使用或重写。
>
>      因此我们可以使用 symbol 属性“秘密地”将一些东西隐藏到我们需要的对象中，但其他地方看不到它。
>
>   2. JavaScript 使用了许多系统 symbol，这些 symbol 可以作为 `Symbol.*` 访问。我们可以使用它们来改变一些内建行为。例如，在本教程的后面部分，我们将使用 `Symbol.iterator` 来进行 [迭代](https://zh.javascript.info/iterable) 操作，使用 `Symbol.toPrimitive` 来设置 [对象原始值的转换](https://zh.javascript.info/object-toprimitive) 等等。
>
> - 从技术上说，symbol 不是 100% 隐藏的。有一个内建方法 [Object.getOwnPropertySymbols(obj)](https://developer.mozilla.org/zh/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertySymbols) 允许我们获取所有的 symbol。还有一个名为 [Reflect.ownKeys(obj)](https://developer.mozilla.org/zh/docs/Web/JavaScript/Reference/Global_Objects/Reflect/ownKeys) 的方法可以返回一个对象的 **所有** 键，包括 symbol。所以它们并不是真正的隐藏。但是大多数库、内建方法和语法结构都没有使用这些方法。

### 应用场景

### React中symbol的应用

https://overreacted.io/why-do-react-elements-have-typeof-property/

你觉得你在写 JSX：

```jsx
<marquee bgcolor="#ffa7c4">hi</marquee>
```

其实，你在调用一个方法：

```jsx
React.createElement(
  /* type */ 'marquee',
  /* props */ { bgcolor: '#ffa7c4' },
  /* children */ 'hi'
)
```

之后方法会返回一个对象给你，我们称此对象为React的 *元素*（element），它告诉 React 下一个要渲染什么。你的组件（component）返回一个它们组成的树（tree）。

```jsx
{
  type: 'marquee',
  props: {
    bgcolor: '#ffa7c4',
    children: 'hi',
  },
  key: null,
  ref: null,
  $$typeof: Symbol.for('react.element'), // 🧐是谁
}
```

如果你用过 React，对 `type`、 `props`、 `key`、 和 `ref` 应该熟悉。 **但 `$$typeof` 是什么？为什么用 `Symbol()` 作为它的值**？

为了避免xss攻击，React等UI库会默认进行文本转义。把潜在危险字符（`<`、`>`等）替换掉。

```jsx
<p>
  {message.text}
</p>
```

如果 `message.text` 是一个带有 `<img>` 或其他标签的恶意字符串，（类似 `'<img src onerror="stealYourPassword()">'`）它不会被当成真的 `<img>` 标签处理，React 会先进行转义 *然后* 插入 DOM 里。所以 `<img>` 标签会以文本的形式展现出来。

转义文本这第一道防线可以拦下许多潜在攻击，知道这样的代码是安全的就够了吗？

**也不总是有效的**。这就是 `$$typeof` 的用武之地了。

React 元素（elements）是设计好的 *plain object*：

```jsx
{
  type: 'marquee',
  props: {
    bgcolor: '#ffa7c4',
    children: 'hi',
  },
  key: null,
  ref: null,
  $$typeof: Symbol.for('react.element'),
}
```

虽然通常用 `React.createElement()` 创建它，但这不是必须的。

但是，如果你的服务器有允许用户存储任意 JSON 对象的漏洞，而前端需要一个字符串，这可能会发生一个问题：

```jsx
// 服务端允许用户存储 JSON
let expectedTextButGotJSON = {
  type: 'div',
  props: {
    dangerouslySetInnerHTML: {
      __html: '/* 把你想的搁着 */'
    },
  },
  // ...
};
let message = { text: expectedTextButGotJSON };

// React 0.13 中有风险
<p>
  {message.text}
</p>
```

在这个例子中，React 0.13[很容易](http://danlec.com/blog/xss-via-a-spoofed-react-element)受到 XSS 攻击。再次声明，**这个攻击是服务端存在漏洞导致的**。不过，React 会为了大家的安全做更多工作。从 React 0.14 开始，它做到了。

React 0.14 修复手段是用 Symbol 标记每个 React 元素（element）：

```jsx
{
  type: 'marquee',
  props: {
    bgcolor: '#ffa7c4',
    children: 'hi',
  },
  key: null,
  ref: null,
  $$typeof: Symbol.for('react.element'),
}
```

这是个有效的办法，因为JSON不支持 `Symbol` 类型。**所以即使服务器存在用JSON作为文本返回安全漏洞，JSON 里也不包含 `Symbol.for('react.element')`**。React 会检测 `element.$$typeof`，如果元素丢失或者无效，会拒绝处理该元素。

特意用 `Symbol.for()` 的好处是 **Symbols 通用于 iframes 和 workers 等环境中**。因此无论在多奇怪的条件下，这方案也不会影响到应用不同部分传递可信的元素。同样，即使页面上有很多个 React 副本，它们也 「接受」 有效的 `$$typeof` 值。

## 迭代器

