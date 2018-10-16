---
title: JS获取各种常见位置距离
date: 2018-03-02 15:03:48
categories: [JavaScript, 基础]
tags: [JavaScript, 前端基础, 位置]
---

## client/offset/scroll系

### width/height

#### element.clientWidth/Height

按照以下判定规则：

1. 如果没有CSS布局盒子或者CSS布局盒子为inline，返回0；
2. 如果是根元素（html元素）或body元素，clientWidth/Height指的是视口的宽高（不含滚动条）
3. 否则，如上，是不包含border不包含滚动条的元素宽高。

显然，其实就是致力于表示元素能够用于显示内容的那部分区域的宽高，所以才叫做clientWidth/Height吧。

**用盒子模型表达，就是content-box去除滚动条占位。**

![clientWidth/Height](http://wx3.sinaimg.cn/large/7b1152ffly1foyj84wh62j20bf06vaa2.jpg)

#### element.offsetWidth/Height

与clientWidth/Height致力于表示元素能显示内容的区域的宽高不同，offsetWidth/Height致力于表示元素本身的宽高。

**用盒子模型表达，即border-box的宽高。**

![offsetWidth/Height](http://wx3.sinaimg.cn/mw690/7b1152ffly1foyj8a47qnj20bf06vdfu.jpg)

#### element.scrollWidth/Height

表示元素内容的宽高，即包括了被卷去部分的宽高。

**用盒子模型表达，就是……没有专门的框，就是显示在content-box中的内容的宽高（内容仅包含常规流中的元素），包括因设置overflow: auto/hidden藏起的部分**

<!-- more -->

### left/top

#### element.clientLeft/Top

用来表示元素左/上边框的宽度。没想到仅仅是用来表达这个意思的，看MDN还以为自己没领会意思，试了下还真是这样……

不过仔细想想也能领会其思想，client指的是能够显示内容区域的部分，clientLeft本身想要指代的是这个区域距离元素本身边界（即border-box）的距离。考虑到值只有clientLeft/Top，而滚动条都在右边或下方，所以滚动条的宽度不被考虑在内（否则一定会算上滚动条的宽度），这个距离中只可能出现border。因此，这个距离就成为了border的宽度。

**用盒子模型表达，就是left/top border width。**

#### element.offsetLeft/Top

##### offsetParent

1. 如果元素为fixed定位，offsetParent为null；
2. 如果元素有最近的有定位的祖先元素，即position不为static的元素，则offsetParent为该元素；
3. 否则，offsetParent为body；
4. body的offsetParent为null。

##### offsetLeft/Top

元素左/上外边框距离offsetParent左/上内边框的距离。

fixed定位元素的offsetLeft/Top为与body内边框的距离。
body的offsetLeft/Top为0。

#### element.scrollLeft/Top

网页向左/上卷去部分的长度。

除了该属性，以上全部属性均为只读。

## element.getBoundingClientRect()

### left/top/right/bottom

元素外边框距离视口左上角(0, 0)点的距离。

![element.getBoundingClientRect().left/right/top/bottom](http://wx4.sinaimg.cn/mw690/7b1152ffly1foyl5m0opwj20dw0dwt94.jpg)

### width/height

width：元素右外边框距离(0, 0)点的距离减去元素左外边框距离(0, 0)点的距离；
height：元素下外边框距离(0, 0)点的距离减去元素上外边框距离(0, 0)点的距离。

## window系

### width/heigth

#### window.innerWidth/Height

浏览器视口的宽高，包括滚动条。

与document.documentElement.clientWidth/Height与document.body.clientWidth/Height的区别是，clientWidth/Height不包含滚动条，而innerWidth/Height包含滚动条。

#### window.outerWidth/Height

浏览器窗口的宽高，包括菜单栏和边框。

#### window.screen.width/height

返回屏幕的分辨率。

#### window.screen.availWidth/Height

返回屏幕的可用分辨率，去除操作系统某些功能占据的空间，如系统任务栏。

### 距离

#### window.screenX/Y

视口左上角相对于屏幕左上角的距离。

#### window.pageXOffset/pageYOffset

页面向左/上卷去的长度。

```
// 取网页卷去距离
const scrollTop = document.documentElement.scrollTop || window.pageYOffset || document.body.scrollTop
const scrollLeft = document.documentElement.scrollLeft || window.pageXOffset || document.body.scrollLeft
```

### [#操作](#操作)操作

#### [#window-scrollTo-x-y](#window-scrollTo-x-y)window.scrollTo(x, y)

将网页滚动到指定位置。

#### [#window-scrollBy-x-y](#window-scrollBy-x-y)window.scrollBy(x, y)

将网页滚动指定距离，即相对位移。如window.scrollBy(0, window.innerHeight)，将网页向下滚动一屏。
