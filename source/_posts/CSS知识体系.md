---
title: css知识体系总结
date: {{ date }}
tags: CSS
cover: https://ppoffice.github.io/hexo-theme-icarus/gallery/covers/vector_landscape_2.svg
excerpt: A total review of css
toc: true
---
## 盒模型

## 选择器、权重

https://zhuanlan.zhihu.com/p/41604775

![img](/images/v2-b1a9fedf320754acb1d7766c6548d5f6_720w.jpg)

```
!important>行内样式>ID选择器 > 类选择器 | 属性选择器 | 伪类选择器 > 元素选择器
```

- 行内样式总会覆盖外部样式表的任何样式(除了!important)

- 单独使用一个选择器的时候，不能跨等级使css规则生效

- 如果两个权重不同的选择器作用在同一元素上，**权重值高**的css规则生效

- 如果两个相同权重的选择器作用在同一元素上：以**后面出现的**选择器为最后规则.

- 权重相同时，与元素距离近的选择器生效



https://juejin.cn/post/7073774752606715935

https://juejin.cn/post/6936913689115099143



https://github.com/cttin/cttin.github.io/issues/2

## margin重叠/合并问题

### 什么是margin collapse

[`W3C`对于外边距叠加的定义](https://link.juejin.cn?target=https%3A%2F%2Fwww.w3.org%2FTR%2FCSS2%2Fbox.html%23collapsing-margins)：

> In CSS, the adjoining margins of two or more boxes (which might or might not be siblings) can combine to form a single margin. Margins that combine this way are said to collapse, and the resulting combined margin is called a collapsed margin.

大概意思是： 在CSS中，两个或多个**毗邻**的**普通流中**的盒子（可能是**父子**元素，也可能是**兄弟**元素）在**垂直**方向上的外边距会发生叠加，这种形成的外边距称之为外边距叠加。

> Horizontal margins never collapse.

只有**垂直**方向的外边距会发生外边距叠加。水平方向的外边距不存在叠加的情况。

### 为什么会有margin collapse

CSS1.0中的规定写的很明白，是故意这样设计的，因为这样在大多数情况下符合平面设计师的要求。

https://zhuanlan.zhihu.com/p/30168984

https://juejin.cn/post/7025953085856235556

### 什么情况会发生margin collapse?

> Two margins are *adjoining* if and only if:
>
> - both belong to in-flow [block-level boxes](https://link.juejin.cn?target=https%3A%2F%2Fwww.w3.org%2FTR%2FCSS2%2Fvisuren.html%23block-boxes) that participate in the same [block formatting context](https://link.juejin.cn?target=https%3A%2F%2Fwww.w3.org%2FTR%2FCSS2%2Fvisuren.html%23block-formatting)
> - no line boxes, no clearance, no padding and no border separate them (Note that [certain zero-height line boxes](https://link.juejin.cn?target=https%3A%2F%2Fwww.w3.org%2FTR%2FCSS2%2Fvisuren.html%23phantom-line-box) (see [9.4.2](https://link.juejin.cn?target=https%3A%2F%2Fwww.w3.org%2FTR%2FCSS2%2Fvisuren.html%23inline-formatting)) are ignored for this purpose.)
> - both belong to vertically-adjacent box edges, i.e. form one of the following pairs:
>   - top margin of a box and top margin of its first in-flow child
>   - bottom margin of box and top margin of its next in-flow following sibling
>   - bottom margin of a last in-flow child and bottom margin of its parent if the parent has 'auto' computed height
>   - top and bottom margins of a box that does not establish a new block formatting context and that has zero computed ['min-height'](https://link.juejin.cn?target=https%3A%2F%2Fwww.w3.org%2FTR%2FCSS2%2Fvisudet.html%23propdef-min-height), zero or 'auto' computed ['height'](https://link.juejin.cn?target=https%3A%2F%2Fwww.w3.org%2FTR%2FCSS2%2Fvisudet.html%23propdef-height), and no in-flow children

- 都属于普通流的块级盒子且在同一个BFC中
- 没有被`padding`、`border`、`clear`和`line box`分隔开
- 在垂直方向上是毗邻的，包括以下几种情况：
  - 盒子的`top margin`和它第一个普通流子元素的`top margin`
  - 盒子的`bottom margin`和它下一个普通流兄弟的`top margin`
  - 盒子的`bottom margin`和它父元素的`bottom margin`
  - 盒子的`top margin`和`bottom margin`，且没有创建一个新的块级格式上下文，且有被计算为0的`min-height`，被计算为0或`auto`的`height`，且没有普通流子元素

### margin collpase 后的大小

当两个外边距都是正数时，取两者之中的较大者；

当两个外边距都是负数时，取两者之间*绝对值*较大者；

当两个外边距一正一负时，取的是两者的和。

## BFC

### 提出BFC的历史背景

为什么在CSS中有Margin collapse,BFC,Containing Block？

https://zhuanlan.zhihu.com/p/30168984

### 什么是BFC？

w3c是这么解释的：

>  浮动元素和绝对定位元素，非块级盒子的块级容器（例如 inline-blocks, table-cells, 和 table-captions），以及overflow值不为“visiable”的块级盒子，都会为他们的内容创建新的`块级格式化上下文`(block formatting context)。

请注意，BFC并不是一个css属性，也不是一段代码，而是css中基于box的一个布局对象，它是页面中的一块渲染区域，并且有一套渲染规则，它决定了其子元素将如何定位，以及和其他元素的关系和相互作用。明确地，它是一个独立的盒子，并且这个独立的盒子内部布局不受外界影响，当然，BFC也不会影响到外面的元素。

### 成为BFC的条件

其实在w3c规范中已经简单罗列了称为BFC的基本条件，但我们还是详细说明下，保持记忆脉络清晰。

一个BFC是一个HTML盒子并且至少满足下列条件中的任何一个：

1. `float`的值不为**none**
2. `position`的值不为**static**或者**relative**
3. `display`的值为 **table-cell**, **table-caption**, **inline-block**,**flex**, 或者 **inline-flex**中的其中一个
4. `overflow`的值不为**visible**
5. 根元素

### BFC的特性

BFC的特性可以总结为以下几点：

1. BFC内部，盒子由上至下按顺序进行排列，其间隙由盒子的外边距决定，并且，当**同一个BFC**中的两个盒子同时具有相对方向的外边距时，其外边距还会发生**叠加**(Margin Collapse)
2. BFC内部，无论是浮动盒子还是普通盒子，其左**总是**与包含块的左边相接触
3. BFC 区域**不会**与float box区域相叠加
4. BFC内外布局**不会**相互影响
5. 计算BFC高度的时候，浮动元素的高度也计算**在内**

### 触发BFC

根据成为BFC的条件，一般有以下4种方法触发BFC：

1. display: table `前后带有换行符，我们一般也不常用`
2. overflow: scroll `可能会出现不想要的滚动条，丑`
3. float: left `万一我们不想让元素浮动呢？`
4. overflow: hidden `比较完美的创建BFC的方案，副作用较小`

### BFC能解决的问题

#### 1. 解决margin collapse(外边距叠加)

什么是外边距叠加？复述一遍：

> 当**同一个BFC**中的两个盒子同时具有相对方向的外边距时，外边距会发生**叠加**(Margin Collapse)，当兄弟盒子的外边距不一样时，将以最大的那个外边距为准。

![image-20220705143439343](/images/image-20220705143439343.png)

注意到这也是只有处于同一BFC才会发生的问题。

要解决上述问题，设想：**如果他们属于不同的BFC，他们之间的外边距将不会折叠**。我们给其中的一个盒子外部包裹一层父盒子，并成为一个BFC。

```html
<div class="container">
  <div class="bfc">
    <div class="part">
    </div>
  </div>
  <div class="part">
  </div>
  <div class="part">
  </div>
</div>
```

```css
.bfc {
  overflow: hidden;
}
.container {
  border: 1px solid red;
  width: 200px;
  margin: 0 auto;
}
.part {
  height: 20px;
  background: #bcbcbc;
  width: 100px;
  margin: 20px auto;
}
```

结果

![image-20220705143344785](/images/image-20220705143344785.png)

#### 2.解决float导致父元素高度塌陷的问题

我们往往会遇到这样一种情况，父元素高度自适应，子元素浮动，因为不在同一文档流中，父元素的高度会坍塌，如下所示：

```html
<div class="container">
  <div class="float">
  </div>
  <div class="float">
  </div>
</div>
```

```css
.container {
  border: 1px solid red;
  width: 200px;
  margin: 0 auto;
}
.float {
  height: 20px;
  background: #bcbcbc;
  width: 80px;
  margin: 10px;
  float: left;
}
```

![image-20220705144118950](/images/image-20220705144118950.png)



这个时候，我们可以利用BFC来解决问题，根据是：`计算BFC高度的时候，浮动元素的高度也计算在内`

我们通过给父元素添加`overflow: hidden`，在容器中创建一个新的BFC，效果如下：

```css
.container {
  border: 1px solid red;
  width: 200px;
  margin: 0 auto;
  overflow: hidden; // 父元素创建BFC
}
.float {
  height: 20px;
  background: #bcbcbc;
  width: 80px;
  margin: 10px;
  float: left;
}
```

![image-20220705144127974](/images/image-20220705144127974.png)

#### 3.实现两栏布局

一般的后台管理系统，很多都是传统的左菜单右内容的两栏布局，我们经常会选择左栏浮动，右边设置左padding或者margin的思路来实现这一做法，其实利用BFC也可以创建两栏布局，根据是：`BFC 区域不会与float box区域相叠加`

```html
<div class="container">
  <div class="aside">
    aside
  </div>
  <div class="main">
    main content
  </div>
</div>
```

```css
.container {
  border: 1px solid red;
  width: 300px;
  margin: 0 auto;
  overflow: hidden;
}
.aside {
  text-align: center;
  height: 150px;
  background: #bcbcbc;
  width: 100px;
  margin: 10px;
  float: left;  // 左边float
}
.main {
  text-align: center;
  background: #abcded;
  overflow: hidden;  // 右边创建一个BFC，不会与左侧float重叠
  height: 150px;
  margin: 10px;
}
```



## IFC

## Flex布局

### flex三个属性的意义

1. flex-gow
   - 定义项目的放大比例，默认为0，即如果存在剩余空间，也不放大。

```css
flex-grow: number | 0;
// 0 为0则表示剩余空间不重新分配；(默认值)
// number 为正整数则按比例分配多余空间
```

2. flex-shrink

- 定义了项目的缩小比例，默认为1，即如果空间不足，该项目将缩小

```css
flex-shrink: 1 | 0
// 1 空间不足时将缩小
// 0 空间不足时不缩小
```

3. flex-basis

- 给上面两个属性分配多余空间之前，计算项目是否有多余空间，默认值为auto,即项目本身的大小

```css
flex-basis: number | auto;
// number  一个长度单位或者一个百分比，规定灵活项目的初始长度。
// auto  长度等于灵活项目的长度。如果该项目未指定长度，则长度将根据内容决定。(默认值)
```


flex属性是 `flex-grow`、`flex-shrink`、`flex-basis`的简写，默认为 0，1，auto。

### flex缩写的使用场景

常见的flex缩写有下面这几个，`flex:0`、`flex:1`、`flex:none`、`flex:auto`

![image](/images/123053883-eded3380-d436-11eb-8dba-8a836eb7bdb7.png)

#### flex: 1

当希望元素充分利用剩余空间，同时不会侵占其他元素应有的宽度的时候，适合使用flex:1，这样的场景在Flex布局中非常的多。

例如所有的**等分**列表，或者**等比例**列表都适合使用flex:1或者其他flex数值



#### flex:none

flex:none等同于设置flex: 0 0 auto

**适合使用flex:none的场景**

当flex子项的宽度就是内容的宽度，且内容永远不会换行，则适合使用flex:none，这个场景比flex:0适用的场景要更常见。

例如列表右侧经常会有一个操作按钮，对于按钮元素而言，里面的文字内容一定是不能换行的，此时，就非常适合设置flex:none
[![image](/images/123062543-34df2700-d43f-11eb-8494-1092cfb775f3.png)](https://user-images.githubusercontent.com/22321620/123062543-34df2700-d43f-11eb-8494-1092cfb775f3.png)



## Grid布局

`flex` 布局和 `Grid` 布局有实质的区别，那就是 **`flex` 布局是一维布局，`Grid` 布局是二维布局**。`flex` 布局一次只能处理一个维度上的元素布局，一行或者一列。`Grid` 布局是将容器划分成了“行”和“列”，产生了一个个的网格，我们可以将网格元素放在与这些行和列相关的位置上，从而达到我们布局的目的。




## 清除浮动的原理

## 布局

### 两栏

### 三栏

### 圣杯

### 双飞翼

## CSS如何实现固定宽高比

https://juejin.cn/post/6844904070679887886

如果元素的尺寸已知的话，没什么好说的，计算好宽高写上去就行了。

如果元素尺寸未知，最简单的方法是用 JavaScript 实现，如果用 CSS 的话又要分为以下几种：

- 如果是可替换元素`<img>`或`<video>`，可以将`width`/`height`其一设定尺寸，另一个设为`auto`，则可替换元素会根据其固有尺寸进行变化。
- 如果是普通的元素，我们可以通过`padding-top`/`padding-bottom`设为百分比的方式来模拟固定宽高比，（**垂直方向上的内外边距使用百分比做单位时，是基于包含块的宽度来计算的**。）不过这种方式不灵活，只能够高度随着**宽度**变。CSS 工作组现在正在引入一种新的方案`aspect-ratio`，可以很方便地指定宽高比，不过暂时还没有浏览器实现。相信不久之后就会有浏览器逐渐实现了。

`aspect-ratio`的语法格式如下：`aspect-ratio: <width-ratio>/<height-ratio>`。

如下，我们可以将`width`或`height`设为`auto`，然后指定`aspect-ratio`。另一个值就会按照比例自动变化。

```css
/* 高度随动 */
.box1 {
  width: 100%;
  height: auto;
  aspect-ratio: 16/9;
}
/* 宽度随动 */
.box2 {
  height: 100%;
  width: auto;
  aspect-ratio: 16/9;
}
```


