---
title: Flex布局
date: 2017-05-08 16:52:15
categories: [CSS]
tags: [CSS]
---

## 基本概念

Flex是Flexible Box的缩写，意为“弹性布局”。设为Flex布局以后，子元素的float、clear和vertical-align属性将失效。

```
// 任何容器都可以指定为Flex布局
.box {
  display: flex;
}

// 行内元素也可以使用Flex布局
.box {
  display: inline-flex;
}
```

采用Flex布局的元素，称为Flex容器（flex container），简称“容器”。它的所有子元素自动成为容器成员，称为Flex项目（flex item），简称“项目”。

容器默认存在两个轴，水平的主轴（main axis）和垂直的交叉轴（cross axis）。主轴的开始位置（与边框的交叉点）叫做`main start`，结束为止叫做`main end`；交叉轴的开始位置叫做`cross start`，结束位置叫做`cross end`。

项目默认沿主轴排列。单个项目占据的主轴空间叫做`main size`，占据的交叉轴空间叫做`cross size`。

<!-- more -->

## 容器的属性

- flex-direction
- flex-wrap
- flex-flow
- justify-content
- align-items
- align-content

### flex-direction属性

`flex-direction`属性决定主轴的方向（即项目的排列方向），它可能有四个值：

- row（默认值）：主轴为水平方向，起点在左端；
- row-reverse: 主轴为水平方向，起点在右端；
- column：主轴为垂直方向，起点在上沿；
- column-reverse：主轴为垂直方向，起点在下沿。

### flex-wrap属性

默认情况下，项目都排列在一条线（又称“轴线”）上。`flex-wrap`定义，如果一条轴线排不下，如何换行。

- nowrap（默认）：不换行；
**以主轴水平方向为例，如果设置为nowrap，项目超出部分不会超出flex容器，而是所有项目等比例缩减宽度，达到正好占满主轴的效果（看到后面发现，这是因为项目的默认flex-shrink属性值为1）**
![no-wrap](http://wx3.sinaimg.cn/large/7b1152ffly1ffe8yf35ygj20jg041aam.jpg)
- wrap：换行，第一行在上方；
**如果设置为换行，则正常宽度，正常换行**
![wrap](http://wx3.sinaimg.cn/large/7b1152ffly1ffe8yi0oipj20jg04x3yv.jpg)
- wrap-reverse：换行，第一行在下方。
![wrap-reverse](http://wx3.sinaimg.cn/mw690/7b1152ffly1ffe8ykzovcj20jg04xmxf.jpg)

### flex-flow属性

`flex-flow`属性是`flex-direction`属性和`flex-wrap`属性的简写形式，默认值为`row nowrap`。

### justify-content属性

`justify-content`属性定义了项目在主轴上的对齐方式。它可能取五个值，具体对齐方式与轴的方向有关。下面假设主轴为从左到右：

- flex-start（默认值）：左对齐；
- flex-end：右对齐；
- center：居中；
- space-between：两端对齐，项目之间的间隔都相等；
- space-around：每个项目两侧的间隔相等。所以，项目之间的间隔比项目与边框的间隔大一倍。
![justify-content](http://wx1.sinaimg.cn/mw690/7b1152ffly1ffe9g4ci6zj20hp0l70sn.jpg)

### align-items属性

`align-items`属性定义了项目在交叉轴上的对齐方式。它可能取五个值，具体对齐方式与轴的方向有关。下面假设交叉轴从上到下：

- flex-start：交叉轴的起点对齐；
- flex-end：交叉轴的终点对齐；
- center：交叉轴的中点对齐；
- baseline：项目的第一行文字的基线对齐；
- stretch（默认值）：如果项目未设置高度或设为auto，将占满整个容器高度。

### align-content属性

`align-content`属性定义了多根轴线的对齐方式。如果项目只有一根轴线，该属性不起作用。

- flex-start：与交叉轴的起点对齐；
- flex-end：与交叉轴的终点对齐；
- center：与交叉轴的中点对齐；
- space-between：与交叉轴两端对齐，轴线之间的间隔平均分布；
- space-around：每根轴线两侧的间隔都相等。所以，轴线之间的间隔比轴线与边框的间隔大一倍；**经测，“轴线之间的间隔”指的实际上是一条轴线上最突出的项目与相邻轴线上最突出的项目的间隔**
- stretch（默认值）：轴线占满整个交叉轴。

## 项目的属性

- order
- flex-grow
- flex-shrink
- flex-basis
- flex
- align-self

### order属性

`order`属性定义项目的排列顺序。数值越小，排列越靠前，默认为0。

### flex-grow属性

`flex-grow`属性定义项目的放大比例，默认为0，即即使存在剩余空间，也不放大。（交叉轴默认占满所有空间，主轴默认不占满。）

如果所有项目的`flex-grow`属性都为1，它们将等分剩余空间（如果有的话）。如果一个项目的`flex-grow`属性为2，其他都为1，则前者占据的剩余空间将比其他项大一倍。
![flex-grow](http://wx1.sinaimg.cn/mw690/7b1152ffly1fff9d0r744j20ma05v3yf.jpg)

### flex-shrink属性

`flex-shrink`属性定义了项目的缩小比例，默认为1，如果项目空间不足，该项目将缩小。

如果所有项目的属性为1，在空间不足时，都将等比例缩小。如果一个项目的flex-shrink为0，其他为1，则空间不足时，前者不缩小。

负值对该属性无效。

### flex-basis属性

`flex-basis`定义了在分配多余空间之前，项目占据的主轴空间。浏览器根据这个属性，计算主轴是否有多余空间。它的默认值为auto，即项目本来的大小。

它可以设为和width或height属性一样的值（如350px），则项目将占据固定空间。

### flex属性

`flex`属性是`flex-grow`、`flex-shrink`和`flex-basis`的简写，默认为0 1 auto。后两个属性可选。

该属性有两个快捷值，auto（1 1 auto，即所有项目等分主轴）和none（0 0 auto，即所有项目保持不变，表现得仿佛项目大小不受Flex布局影响）。

优先写这个属性，而不是单独写三个分离的属性，因为浏览器会推算相关值。

### align-self属性

`align-self`属性允许单个项目有与其他项目不一样的对齐方式，可覆盖align-items属性。默认值为auto，表示继承父元素的align-items属性，如果没有父元素，等同于stretch。

该属性可能取六个值，除了auto，其他都与align-items属性完全一致。

## flex-grow&flex-shrink详细计算规则

### flex-grow

定义项目的放大比例，默认值为0，即存在剩余空间也不放大。

flex-grow的默认值为0，如果没有定义该属性，是不会拥有分配剩余空间的权利的。A, B, C, D, E 的宽度分别是 100, 120, 130, 100, 100，父级的宽度是660，父级宽减去子级的全部宽度，这样剩余空间就是110。例子中B, C定义了flex-grow，分别是1，2，那剩余空间分成3份，B1份，C2份，比例就是1：2，分配计算出来的值就是B :36.666666666666664, C：73.33333333333333。这个时候剩余空间就被计算出来了，相加后的结果就是B：156.66666666666666，C：203.33333333333331。
B的计算公式：120 + (110 / 3) × 1。
C的计算公式: 130 + (110 / 3) × 2。

### flex-shrink

定义项目的缩小比例，默认为1，即空间不足则按比例缩小。

这里A、B、C的宽度分别为155、200、50。父级宽度是300，计算超出宽度就为-105，这样超出的105px就要被消化掉，比例是2:1:1。
消化方法，首先每个项目的宽度乘flex-shrink值求积：
A: 155 × 2 = 310
B: 200 × 1 = 200
C: 50 × 1 = 50

然后求所有乘积的和：
310 + 200 + 50 = 560

之后按照占比分配需要腾出的空间：
A: 310 / 560 × 105 = 58.125
B: 200 / 560 × 105 = 37.5
C: 50 / 560 × 105 = 9.375

得出项目的最终宽度：
A: 155 - 58.125 = 96.875
B: 200 - 37.5 = 162.5
C: 50 - 9.375 = 40.625

### 总结

flex-basis和width作用一样，优先级更高，更规范。
如果父级空间足够，flex-grow有效，flex-shrink无效；
如果父级空间不足，flex-shrink有效，flex-grow无效。

## 参考文献

- 阮一峰，[Flex 布局教程：语法篇](http://www.ruanyifeng.com/blog/2015/07/flex-grammar.html?utm_source=tuicool)
- [深入理解 flex-grow & flex-shrink & flex-basis](https://xiecg.github.io/2016/08/28/flex/)
