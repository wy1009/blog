---
title: CSS零碎
date: 2017-03-08 16:35:15
categories: [CSS]
tags: [CSS]
---

## 盒模型在W3C和IE中的区别

IE盒模型问题只会出现在IE5.5及更早的版本。

W3c盒模型，width = content-width。设定好width后，增加padding等都会增加整体盒子的宽度。

IE盒模型，width = content-width + padding + border。设定好width后，增加padding会减少content-width的宽度，整体盒子的宽度不变。

## EM

父元素也用em表示，子元素font-size如何计算：子元素以父元素font-size为基准，子元素1em等于父元素font-size大小。若子元素设为1.2em，实际px为1.2*1.2。

## 绝对定位，水平竖直居中，需要设定margin: auto的原因

水平竖直居中有一种写法，设定宽高，left、right、top、bottom都为0，margin为auto。

绝对定位的元素的布局取决于三个因素：元素的位置、元素的尺寸、元素的margin。

在元素位置固定的时候则分配尺寸（因此四边都为零会平铺满），尺寸固定则分配margin，此时margin设为auto自适应，元素就居中了。

<!-- more -->

## clear:both之后

clear:both之后，浮动元素下方的元素的margin-top会被覆盖（直到margin-top大到足以顶到一个不浮动的元素），但是浮动元素的margin-bottom却不会被覆盖。

## 关于外边距合并

竖直方向两个元素的margin重合就会“失去效力”。具体表现为，如果是兄弟元素重合，则取大的那个margin。如果是父子元素重合，子元素的margin将不能够“顶开”父元素，而是与父元素（或祖先元素）的margin重合，且仍旧是只有更大的那个生效。

解决方法：

1. 给父元素设置padding/border，有了支撑，子元素的margin将不会脱出。
2. 使父元素脱离文档流，如positon: absolute/fixed、float。之所以只让父元素脱离文档流而不是子元素，是因为父元素脱离文档流，子元素仍在其中支撑出空间。而子元素脱离文档流，则与父元素完全“失联”（除了定位），父元素则失去了支撑，完全塌陷。
3. 父元素overflow: hidden/auto。
