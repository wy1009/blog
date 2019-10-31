---
title: 一个比较有趣的需求
date: 2017-10-18 18:09:58
categories: [框架/库/工具, React.js]
tags: [记录]
---

因为比较少见，是难得比较有趣的需求，记录一下。

首先介绍一下需求。

<img src="/images/complex-01.jpeg" style="width: 300px;" />
<img src="/images/complex-03.png" style="width: 300px;" />

大概就是要实现如图效果，需求点拆分大概如下：

1. 给出一个句子，使用户可以在句子上划动，选中句子中的一部分。可以取消选中。
2. 将选中部分的位置（下称 index）提交给后端。不能单单提交选中的内容，因为同一句中可能会出现重复的单词。
3. 给出 index，可以将对应 index 转化为横线，划在句子上。句子中的下划线还可以再次叠加下划线，叠加的线条数目也是无上限的。

第2、3条合并起来，其实就是 UI 视图到 index，与 index 到视图的相互转化。

<!-- more -->

## 定位用户划动位置，做选中标记

### Range API

关于这个，原本其实是想要用 [Range](https://developer.mozilla.org/en-US/docs/Web/API/Range) api 去解决的。

<img src="/images/complex-02.png" />

可以看到，这个 api 仿佛天生就是为了“标记一段文本”这样的功能存在的。它甚至可以直接从系统原生的 `Select` api 获取对应的 `Range`，让你觉得你连选择过程都不需要自己去写了，使用系统原生就可以。

然而经过评估，这个方法却是无法达成需求的。其中最麻烦的一点莫过于，在选中文本之后，我们必然要将其标记起来。你会发现，一旦做了标记，`Range` api 中天生指定起点终点位置的 `Range.startOffset/endOffset` ——也就是选中文本在整段文本中的起点和终点的 index ——就不再准确了。因为为了做添加背景色之类的标记，必然要添加譬如 `<span>` 的元素，导致 `Range.startContainer/endContainer` 变化，最终导致 `Range.startOffset/endOffset` 不再是我们需要的那个值。

我也曾经考虑过使用两段一模一样的文本重叠在一起，一段专门用来选择并得出对应 index，另一段专门用于显示背景色标记。但考虑到一旦专门用于显示标记的那段文本被做过了标记，那么得出 index 就同样变得不容易插入到正确的位置，与上面的问题如出一辙，因此也作罢了。

同时，利用该方法还存在着没有额外的确定按钮，因而不容易确定用户何时真正想要结束选择状态等问题。

考虑到利用 `Range` api 的优势实际上也只有避免自己写选择过程与自动返回 index，而这两个优势都不是那么容易达成，因此放弃该 api，改为由自己实现。

### 分词 + touch 事件

#### 分词

最终确定的解决方案，是先将一段文本分词。

<img src="/images/complex-15.png" />

分词替换之后，一段文本就变成了下图的样子：

<img src="/images/complex-04.png" />

所做的操作，其实就是将整段文本拆分为单词/空格/标点符号等维度，并设置好 `data-index` 标记用于指示当前单词的位置（index），`data-selected` 标记用于指示单词是否被选择。

这样，index 的维度也不再是原定的字符，而是单词。

#### touch 事件

在分词之后，只要在整段文本绑定 touch 事件，就可以实时获取当前触摸到的元素，用以实现单击/划动选中单词的效果。

##### 划动选择

<img src="/images/complex-05.png" />

`selectWord` 方法很简单，给对应单词带上类名以显示被选中的样式，并标记上 `data-selected` 用于指示是否被选中即可：

<img src="/images/complex-06.png" style="width: 600px;" />

相对应的，取消选中也很简单，移除类名，清空 `data-selected` 即可。

其中有几个比较值得注意的点。

第一，`touchmove` 事件的 `e.target` 并不会实时跟着用户手指划过的元素变化，而是一直停留在 `touchstart` 时碰触到的那个元素。为了实时获取用户划过的元素，需要利用当前碰触的坐标求出碰触的元素：

``` JavaScript
const touch = e.targetTouches[0]
const ele = document.elementFromPoint(touch.clientX, touch.clientY)
```

`touch` 坐标的含义详见 [MDN-Touch](https://developer.mozilla.org/en-US/docs/Web/API/Touch)。

第二，划动过程中需要记下本次涉及到的所有单词元素。因为是划动的交互效果，每次涉及到的单词元素必然是一个连续区间。如果当前划动到的单词在这个区间之外，则需要补充“旧区间”加上“旧区间到当前划动到的点”之间的所有单词作为新区间；如果当前划动到的单词在区间之内，则需要将“旧区间起点”到“当前划动到的单词”之间的所有单词作为新区间，舍弃掉多余的，以实现在划动过程中回退可以舍弃已选择单词的效果。

简单说，就是实时计算起点单词到手指触摸单词作为本次区间，标记颜色，并舍弃已不属于当次区间的。所谓“当次”的定义见第三条。

第三，标记单词元素上用于指示该单词是否被选中的属性不是布尔值，而是一个数字。每次选择单词，数字递增，以此来标记被选中的单词是否是当次选中的，即是否处于当前正在被操纵中的状态。在第二步中“舍弃已选择单词”，也只舍弃本次正在操纵中的。之所以这样做，是因为需求同时要求之前的选择可以被保存在文本中，而不是每次选择都重置过去所有选择。

##### 单击选择

<img src="/images/complex-07.png" />

单击选择这里只做了一个特殊处理，即读被选单词只间隔空格的前后两个单词，如果前/后的单词也是被选中的，则将其中的空格也选中。

##### 取消选择

<img src="/images/complex-08.png" style="width: 660px;" />

取消选择的逻辑也很简单，只要以点击取消选择的单词为原点，分别向左右遍历，取消掉与该原点相连且被选中的元素的选中状态即可。

## 将用户的选择转化为 index

<img src="/images/complex-09.png" style="width: 660px;" />

遍历整个单词列表，取出被选中的单词的 index 即可。

## 将 index 转化为下划线

将 index 转化为下划线，要求内容下可以叠加多条下划线，下划线数目无上限，且每次划线都需要变化颜色。为达成这个需求，需要注意三个方面。

### 线的位置

线的位置，是通过数个形如 `[0, 1, 2, 5, 6, 7]` 的列表得出的，每个列表代表着一次划线。

拿到一个列表，首先，需要求出所有连续的数字，得出“区间”。因为你要在某个词组下划线，必然要保证在这个词组中，每个单词的下划线的位置都是相同的，这样才能形成在一个词组下划了线的印象。而需要划线的单词只要相连，形成了区间，我们就可以理解为是一个词组，需要在同样的位置上划线。

得出区间之后，还需要遍历整个区间的单词，找到所有单词中已经被划线最多的单词。比如区间中被划线最多的单词下已经有三条线了，那么再在该区间划线时，就需要统一在第四条线的位置上开始划线。

当然，看到这里，你可能会有些疑惑。如果某条线是在第四条线的位置上划的，但是它对应的单词下其实还只有一条线，那么岂不是就出错了。下一个区间如果最大线数还是三，但是该单词已经在第四条线的位置上划了线，就会造成重叠。

所以，遍历区间“被划线最多的单词”的说法只是方便在刚引入概念时进行解释。实际上，我们遍历的是“被划线位置最高”的元素。也就是说，一个元素下面只有一条线，但是被划在了第四条线的位置上，我们的计数就是 4，而不是 1。

<img src="/images/complex-10.png" />

通过上面的方法，我们就通过一个划线列表求出了几段区间，以及每段区间对应的划线位置。

### 行下间距

确定了线的位置，接下来，我们就需要考虑每行行下间距的问题了。因为行下间距有限，如果需要划线的位置太高，我们就需要扩展行下间距。

<img src="/images/complex-11.png" />

上面实际上是给元素划线的过程。可以看到，如果划线的高度超过一定的数值，就会扩展单词下面的间距。因为原本就是 `inline-block` 的布局，本行其他元素也会随之对齐，扩展出相应的间距。

另外，每个单词划线的高度也是在这里被记录下来的。每次划线都记录下当前的高度 `el.lineCount`，下次划线遍历元素就可以直接遍历该值，取最大值的下一高度开始划线。

顺带一提，从代码上看，划线实际上是通过给单词元素加子元素实现的。所以，重置划线状态的方式也十分简单，不需要刻意操作线条元素，直接使单词元素 `el.innerHTML = el.innerText` 即可，十分方便。`el.innerText` 可以直接取到纯净的单词内容。

### 每次划线变换颜色

只要在每划完一条线之后，将当前颜色置为颜色列表中的下一项即可。到头即回到第一项。

## 组合三大功能

至此，我们已经将三大功能，包括分词、选择单词、给单词划线介绍完了。已知在实际需求中，我们有时只需要其中的一种功能，有时候需要两种，有时候则三种都需要。总不可能把三种功能都写在一起。那么，我们应如何将三种功能优雅地组织在一起呢？

这时候，我们可以想到三种常见的解决方案：React Hooks、render props、HOC。

### React Hooks

众所周知，React Hooks 一直是以“可以复用逻辑”而为人所知的。但是，在当前需求下，我却怎么都无法想出如何用 React Hooks 来满足自己的需要。

以我对 React Hooks 的理解，其的逻辑复用实际上就是，在自定义 Hook 中巴拉巴拉执行完所有操作，包括可增加在特定生命周期钩子中执行的东西，然后吐出一些数据或是一个节点之类，总之是较为单纯而单一的逻辑。可是，我需要实际地对源组件进行方法的扩充，节点的增加等做各种复杂操作，逻辑没有那么单纯，不是吐出一个数据或是一个节点就能完成所有逻辑的。

理论上讲，React Hooks 应该是可以替代所有 HOC 和 render props 的使用场景。可是，如果选择 React Hooks，我想不出如何能够优雅地满足我的需求。

再次研读官方文档：

<img src="/images/complex-12.png" style="width: 600px;" />

简单说吧，我没看懂。可视滚动条的 `renderItem` 是什么意思？容器组件有其自己的 DOM？容器组件指的是 render props 中作为公共组件的那个吗？还是什么其他的“容器组件”？我确实是没有看懂。

### render props

render props 实际上完全可以满足我们的需求的。可是，它调用起来是这个样子的：

<img src="/images/complex-13.png" style="width: 400px;" />

其中，`Content` 指的是用来分词并渲染的组件，`ContentSelect` 是用于添加选择功能的组件，`ContentLine` 是用于添加下划线功能的组件。

可以看到，用这种写法，每次调用都是比较麻烦而难受的。而在我们的项目里有多处需要调用上述几个组件灵活组合的场景，这样无疑多了不少无意义的样板代码。

感觉相比较于 HOC，render props 的场景更倾向于向公共组件填充需要灵活变动的部分。而我们的需求显然并不需要那样灵活。

### HOC

最终，我采用的 HOC 的组合方式。大体结构如下：

<img src="/images/complex-14.png" style="width: 600px;" />

很普通的高阶组件写法。调用起来也十分简单方便：

``` JavaScript
const ContentDivide = withContentSelect(withContentLine(Content))
const ContentLine = withContentLine(Content)

<ContentDivide
  lines={props.preCorrectAnswers}
  content={props.content}
  onSubmit={handleSubmit}
/>
```

可以做到灵活的组合，自由调配自己所需要的功能。