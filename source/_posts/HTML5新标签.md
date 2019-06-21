---
title: HTML5新标签
date: 2017-02-22 15:31:42
categories: [HTML]
tags: [HTML5]
---

## article

article元素表示文档、页面、应用或网站中的独立结构，其意在成为可独立分配的或可复用的结构，如在发布中，它可能是论坛帖子、杂志或新闻文章、博客、用户提交的评论、交互式组件，或者其他独立的内容项目。

* 当元素嵌套使用时，则该元素代表与外层元素有关的文章。例如，代表博客评论的元素可嵌套在代表博客文章的元素中。
* 元素的作者信息可通过address元素提供，但是不适用于嵌套的article元素。
* article元素的发布日期和时间可通过time元素的pubdate属性表示。

``` HTML
<article class="film_review">
  <header>
    <h2>Jurassic Park</h2>
  </header>
  <section class="main_review">
    <p>Dinos were great!</p>
  </section>
  <section class="user_reviews">
    <article class="user_review">
      <p>Way too scary for me.</p>
      <footer>
        <p>
          Posted on <time datetime="2015-05-16 19:00">May 16</time> by Lisa.
        </p>
      </footer>
    </article>
    <article class="user_review">
      <p>I agree, dinos are my favorite.</p>
      <footer>
        <p>
          Posted on <time datetime="2015-05-17 19:00">May 17</time> by Tom.
        </p>
      </footer>
    </article>
  </section>
  <footer>
    <p>
      Posted on <time datetime="2015-05-15 19:00">May 15</time> by Staff.
    </p>
  </footer>
</article>
```

<!-- more -->

## section

section元素表示文档中的一个区域（或节），比如，内容中的一个专题组，一般来说会有包含一个标题（heading）。一般通过是否包含一个标题 (h1-h6元素) 作为子节点来辨识每一个section。

* 如果元素内容可以分为几个部分的话，应该使用article而不是section。
* 不要把section元素作为一个普通的容器来使用，这是本应该是div的用法（特别是当片段（the sectioning ）仅仅是为了美化样式的时候）。 一般来说，一个section应该出现在文档大纲中。

## aside

aside元素表示一个和其余页面内容几乎无关的部分，被认为是独立于该内容的一部分并且可以被单独的拆分出来而不会使整体受影响。其通常表现为侧边栏或者嵌入内容。他们通常包含在工具条，例如来自词汇表的定义。也可能有其他类型的信息，例如相关的广告、笔者的传记、web 应用程序、个人资料信息，或在博客上的相关链接。

* 不要用aside元素去标记括号内的文本，因为这种类型的文本（即括号内的文本）被认为是其主体（mail flow）的一部分。

## audio

audio元素用于在文档中表示音频内容。audio元素可以包含多个音频资源,这些音频资源可以使用src属性或者source元素来进行描述；浏览器将会选择最合适的一个来使用。对于不支持audio元素的浏览器，audio元素也可以作为浏览器不识别的内容加入到文档中。

## video

video元素用于在HTML或者XHTML文档中嵌入视频内容。

``` HTML
<!-- Simple video example -->
<video src="videofile.ogg" autoplay poster="posterimage.jpg">
  抱歉，您的浏览器不支持内嵌视频，不过不用担心，你可以 <a href="videofile.ogg">下载</a>
  并用你喜欢的播放器观看!
</video>
<!-- Video with subtitles -->
<video src="foo.ogg">
  <track kind="subtitles" src="foo.en.vtt" srclang="en" label="English">
  <track kind="subtitles" src="foo.sv.vtt" srclang="sv" label="Svenska">
</video>
```

第一个例子播放一个视频，视频一收到，允许播放的时候就会立马开始播放，而不会停下来直到下载更多内容。 图片 “posterimage.jpg” 会一直展示在视频区域，直到开始播放。

第二个例子允许用户选择不同的字幕。

## canvas

canvas元素可被用来通过脚本（通常是JavaScript）绘制图形。比如,它可以被用来绘制图形,制作图片集合,甚至用来实现动画效果。你可以(也应该)在元素标签内写入可提供替代的的代码内容，这些内容将会在在旧的、不支持canvas元素的浏览器或是禁用了JavaScript的浏览器内渲染并展现。

## command

command元素用来表示一个用户可以调用的命令。根据type属性定义其为单选、复选或按钮。

## datalist

datalist元素包含了一组option元素，这些元素表示其他表单控件元素的可选值。如与input元素配合使用，可以制作输入值的下拉列表。

## details

details元素用于描述文档或文档某个部分的细节，用户可进行查看，或通过点击进行隐藏。与legend元素一起使用，可以制作detail的标题。

## summary

summary元素包含details元素的标题，details元素用于描述有关文档或文档片段的更多详情。

``` HTML
<details>
  <summary>Some details</summary>
  <p>More info about the details.</p>
</details>
```

## embed

embed元素定义嵌入的内容，比如插件。

## figure

figure元素代表一段独立的内容, 经常与说明(caption)figcaption元素配合使用, 并且作为一个独立的引用单元。当它属于主体(main flow)时，它的位置独立于主体。这个标签经常是在主文中引用的图片，插图，表格，代码段等等，当这部分转移到附录中或者其他页面时不会影响到主体。

## figcaption

figcaption元素是与其相关联的图片的说明/标题，用与描述其父节点figure元素里的其他数据。这意味着figcaption在figure块里是第一个或最后一个。同时figcaption元素是可选的；如果没有该元素，这个父节点的图片只是会没有说明/标题。

## header

header元素表示一组引导性的帮助，可能包含标题元素，也可以包含其他元素，像logo、分节头部、搜索表单等。

* header元素不是分节内容，所以不会引入新的分节到大纲中。

## footer

footer元素表示最近一个章节内容或者根节点（sectioning root）元素的页脚。一个页脚通常包含该章节作者、版权数据或者与文档相关的链接等信息。

* footer元素内的作者信息应包含在address元素中。
* footer元素不是分节内容，所以不会引入新的分节到大纲中。

## hgroup

hgroup标签用于对网页或区段（section）的标题进行组合。如：

``` HTML
<hgroup>
  <h1>Main title</h1>
  <h2>Secondary title</h2>
</hgroup>
```

## mark

mark标签代表突出显示的文字,例如可以为了标记特定上下文中的文本而使用这个标签. 举个例子，它可以用来显示搜索引擎搜索后关键词。

## meter

meter元素用来显示已知范围的标量值或者分数值。

``` HTML
<meter value="3" min="0" max="10">3/10</meter>
```

# nav

HTML导航栏 (nav) 描绘一个含有多个超链接的区域，这个区域包含转到其他页面，或者页面内部其他部分的链接列表。

## output

output标签定义一个用户的操作或者计算的结果。

``` HTML
<form oninput="result.value=parseInt(a.value)+parseInt(b.value)">
    <input type="range" name="b" value="50" /> +
    <input type="number" name="a" value="10" /> =
    <output name="result"></output>
</form>
```

## progress

progress元素用来显示一项任务的完成进度。虽然规范中没有规定该元素具体如何显示，浏览器开发商可以自己决定，但通常情况下，该元素都显示为一个进度条形式。

``` HTML
<progress value="70" max="100">70 %</progress>
```

## ruby

ruby元素被用来展示东亚文字注音或字符注释。rt元素显示注音，rp元素定义在当前浏览器不支持ruby元素时所显示的内容。

``` HTML
<ruby>
  漢 <rp>(</rp><rt>Kan</rt><rp>)</rp>
  字 <rp>(</rp><rt>ji</rt><rp>)</rp>
</ruby>
```

## source

source标签为媒介元素（比如video和audio）定义媒介资源。例为拥有两份源文件的音频播放器，浏览器会选择它支持的文件（如果有的话）。

``` HTML
<audio controls>
   <source src="horse.ogg" type="audio/ogg">
   <source src="horse.mp3" type="audio/mpeg">
 Your browser does not support the audio element.
</audio>
```

## time

time标签用来表示24小时制时间或者公历日期，若表示日期则也可包含时间和时区。此元素意在以机器可读的格式表示日期和时间。 有安排日程表功能的应用可以利用这一点。

* 如果给定的日期不是正常日期或者在公历中最早的日期之前，请不要使用此元素。
* 该元素仍在设计和讨论中。
