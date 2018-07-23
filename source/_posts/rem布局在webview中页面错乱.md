---
title: rem布局在webview中页面错乱
date: 2018-07-20 13:13:37
categories: [CSS]
tags: [CSS]
---

查到原因之后，感觉是一个很简单的事，解决方法也很方便很简单，本来没想写什么博客的。可是仔细看了看查到的两篇文章，一篇解决方法写得超级复杂，另一篇就特别……一言难尽。感觉我写这篇博客最主要的原因就是被第二篇文章的方法之差劲给震惊了……

啊，写博客好麻烦好麻烦的……

## 现象及原因

在安卓webview中发现了利用rem布局的样式错乱的问题，以下是正常样式和错乱样式的对比：

![正常样式](/images/rem-webview-right.png)
![错乱样式](/images/rem-webview-error.png)

可以很容易发现，“错乱样式”实际上是rem的基准，即根结点字号，变大导致的。实际上，利用`Chrome://inspect`调试，也能够清楚地看到这一点，设置在根结点上的字号为`font-size: 48px`，而实际调用`getComputedStyle(document.documentElement).fontSize`的字号却是`55.1px`，这样，会产生样式的错乱就一点也不奇怪了。

查询了一下，发现这个问题产生的原因实际上是因为用户设置了系统的字体，影响范围似乎只有安卓app内的webview，测试ios同样的操作并没有产生同样的问题。

## 解决方法

### 前端解决

知道了原因之后，解决方法也变得很容易想到了。

既然基准（根结点字号）变化是因为设置了系统字体大小，那么系统设置一定是将基准按照设置的比例做了放大/缩小。这个比例就是`实际基准 / 设定基准`。我们要做的就是平衡掉这个比例，找出一个新的设定基准，让实际基准能够恰巧我们最初的设定基准。说通俗些，就是按比例放大/缩小，如果想让实际基准能放大/缩小为设计基准，那么按照比例，真正的设计基准是？

由此推出：`实际基准 / 设定基准 = 设定基准 / 新的设定基准`。新的设定基准就等于`设定基准 * 设定基准 / 实际基准`。

<!-- more -->

代码：

```
var computedFontSize = parseFloat(getComputedStyle(document.documentElement).fontSize)
var configFontSize = parseFloat(document.documentElement.style.fontSize)

document.documentElement.style.fontSize = (configFontSize * configFontSize / computedFontSize) + 'px'
```

顺便，当作趣闻讲讲让我写出这篇文章的大哥的写法吧……他也想到了手动调整实际基准，让其等于设定基准。但是这位大哥的做法是，不断循环，一个一个数地减/加，最后试出一个和设置基准相似的实际基准，还担心陷入死循环，只循环一百次……（手动捂脸）

总之……作为程序员，还是……用优雅一点的方法吧……

### 客户端解决

查了下，如果情况是在自家app中内嵌自己的页面，那么这件事客户端也是可以解决的。让他们设置一个默认的webview字体大小，使其不受系统设置影响就可以了。

## 微信IOS版中相似的问题

写到这里，忽然想起在IOS中也遇到过相似的问题。虽然在IOS端调整系统大小没有见到影响rem基准，但是调整IOS版微信的字体大小，会影响到在微信浏览器中的布局。这个当时查到，实际上是在body上设置了`-webkit-text-size-adjust: xx%`。如果实在影响布局，将这个css属性值覆盖为`none`即可。
