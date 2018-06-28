---
title: position、float和display同时作用于一个元素的处理方法
date: 2017-03-30 20:52:05
categories: [CSS]
tags: [CSS]
---

## 相关特性

### 浮动

- 在浮动模型中，一个框（box）首先根据常规流布局，再将它从流中取出并尽可能地向左或向右偏移。因为它首先要根据常规布局后才偏移，所以效率较常规流较低。
- 浮动元素不发生外边距折叠。
- 浮动框的顶边不可以高于源文档中先前元素产生的块框或浮动框的顶。详细如下图，源文件中元素的顺序为O、A、B，O与A左浮动，B右浮动。B右浮动没有在上一行，而是在下一行的原因是，其不可以高于源文件中先前元素产生的块框或浮动框的顶，即不能高过B元素的顶。
![块框或浮动框](http://wx2.sinaimg.cn/large/7b1152ffly1fehr66za2wj205606x3y9.jpg)
- 浮动框的顶边不可以高过源文档中先前元素产生的任何包含一个框的行框的顶。
![行框](http://wx3.sinaimg.cn/large/7b1152ffly1fehrd9t3mlj20bb01k0pq.jpg)
- 在满足以上两条顶边规则的限制下，浮动框要放置得尽可能得高。
- 左浮动框必须尽量向左放置，右浮动框必须尽量向右放置。在更高的位置和更靠左/靠右的位置间，选择前者。
- 附加规则：对于clear特性不是none的浮动框，其上外边界（top margin edge）必须低于前面所有的左浮动框（clear: left）或右浮动框（clear: right）或左右浮动框（clear: both）的下外边界。

#### clear特性

该特性表明一个元素框的哪一边不可以与先前的浮动框相邻。clear特性不考虑它自身包含的浮动子元素和不处于同一格式化上下文中的浮动元素。

- left|right|both：生成框的间隙，是指设置足够的空白区，以使元素的顶边框边界（top border edge）放置到由源文档中较早元素生成的任何左浮动框（left）或右浮动框（right）或左右浮动框（both）的底外边界（bottom outer edge）之下。
- none：对考虑到浮动后的框的位置没有约束。

可以简单地认为设置了clear特性值的元素，其top border edge要放在相关浮动元素的bottom margin edge之下。注意这两种元素接触边界的区别，一个是border，一个是margin。

前面也提到，如果clear特性作用的是一个浮动元素，则规则变成其top margin edge要放在相关浮动元素的bottom margin edge之下。

### 绝对定位

#### position: relative

1. table-row-group、table-header-group、table-footer-group、table-column-group、table-row、table-column、table-cell、table-caption元素的position: relative效果没有被定义。
2. 相对定位元素处于常规流中，没有脱离常规流。position: static——常规流！position: relative——常规流！
3. top、left、right、bottom设置百分比时，top和bottom是依据包含块的高度，left和right是依据包含块的宽度。这点和absolute是一样的。

#### position: absolute

1. 绝对定位元素定位的参照物是其包含块，top、left、right、bottom指定的是元素相对于其包含块的偏移量，因此当包含块为行内元素时，表现会有点颠覆三观。
2. 绝对定位（absolute、fixed）的元素在3D可视化模型中，处于浮动元素的上方，比浮动元素更靠前。

#### 分层显示（Layered presentation）

##### z-index

z-index特性指定层叠级别，只在定位元素（position不为static）上奏效。

-  该整数为生成框在当前层叠上下文中的层叠级别。同时，该框也会生成一个局部层叠上下文，其中它的层叠级别为0。
- auto 生成框在当前层叠上下文中的层叠级别与它的父框相同。**该框不生成新的局部层叠上下文**

**也就是说，如果一个元素z-index为1，与它同一层叠上下文的元素z-index为0，那么为1的元素在为0的元素上方。此时，哪怕为1的元素的子元素z-index为负，那么这个为负的子元素仍在为0的元素上方。因为为1的元素已经生成了一个局部层叠上下文，它是独立的，其中的框不可能出现在其他层叠上下文中。**

##### 层叠上下文的构成

1. 形成层叠上下文的元素的背景和边框
2. 层叠级别为负值的后代层叠上下文
3. 常规流内非行内非定位的子元素形成的层
4. 非定位的浮动子元素和它们的内容组成的层
5. 常规流内行内非定位子元素组成的层
6. 任何z-index为auto的定位子元素，以及z-index是0的层叠上下文组成的层
7. 层叠级别为正值的后代层叠上下文

简单说，就是形成层叠上下文的元素的背景和边框 < z-index为负 < 常规流（非行内） < 浮动 < 常规流行内 < z-index为0/auto < z-index为正。除了行内元素，实际上都很符合常识，只要记得常规流内行内元素高于浮动元素即可。

![层叠上下文的构成](http://wx4.sinaimg.cn/large/7b1152ffly1feisqutl2lj20i10dcwf9.jpg)

既然已经提到了分层，就再复习一下元素盒子模型的切面图吧。

![元素盒子模型切面图](http://wx2.sinaimg.cn/large/7b1152ffly1feisqz4qbyj20dk0crjsz.jpg)

## display、position、float的相互关系

简单说，判断的先后顺序（优先级）就是display:none>position: absolute/fixed>float: left/right>元素是否为根元素（是的话display强行block或table）>display不为none的值。
![](http://wx2.sinaimg.cn/large/7b1152ffly1fe4wzbxzc4j20k00f0t9l.jpg)

设定值对应计算值：

- 设定值inline-table，计算值table；
- 设定值table-header-group、table-footer-group、table-column-group、table-row-group、table-row、table-column、table-cell、table-caption、inline-block、inline、run-in；
- 设定值为其他，则计算值同设定值。

侧面说明，浮动或绝对定位的元素只能是表格或者块级元素，如果不是表格或块级元素，就强行设定为表格或块级元素。

## 参考文献

- W3help, 武利剑, [CSS 定位体系概述](http://www.w3help.org/zh-cn/kb/009/)
