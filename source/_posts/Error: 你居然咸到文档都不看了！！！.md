---
title: "Error: 你居然咸到文档都不看了！！！"
date: 2022-03-08 18:15:36
categories: [框架/库/工具, React.js]
tags: [Next.js, React.js]
---

必须承认，当我决定写 Next 项目时，我并没有先认真学习原理，而是简单看了遍大略文档就直接上了。
但遇到问题发现文档里就有，还是很尴尬的……

## 遇到的问题

在 Next 项目中，通过屏幕宽度确定是否是移动端布局。而在 Server 端生成 HTML 的时候，是无法取到屏幕宽度的。于是，我写了如下代码：

``` JavaScript
// utils.ts
// 是否是移动端布局
const ua = process.browser ? window.navigator.userAgent : ''
export const isMLayout = process.browser && document.documentElement.clientWidth <= 1199

// 页面组件
const Success = () => {
  // Web 端下载部分
  const renderWebDownload = () => (
    <section className={s.downloadWebContainer}>
      // ...
    </section>
  )

  // Wap 端下载部分
  const renderWapDownload = () => {
    return (
      <>
        <section className={s.downloadAppContainer}>
          // ...
        </section>

        <section className={s.downloadPcContainer}>Please go to the Web terminal to view and download.</section>
      </>
    )
  }

  return (
    <article>
      // ...
      {isMLayout ? renderWapDownload() : renderWebDownload()}
      // ...
    </article>
  )
}

export default Success
```

这段代码产生了问题。在 Client 端渲染后，`renderWapDownload` 生成的代码块，最外层的包裹却是 `<section className={s.downloadWebContainer}></section>`，也就是在渲染时复用了一部分 web 端的 DOM。

仔细想想，这也是有迹可循的。用移动端访问该页面时，Server 端吐出的 html 会是 web 端的情况，而在 Client 端收到之后，吐出的 html 会是 wap 端的情况。Server 端生成的 html 和 Client 端生成的 html 会是不同的。这应该是导致 Client 端渲染 wap 样式时，复用了一部分 web DOM 的原因。

有趣的是，我专门写了一个 state，用来促使页面重新渲染，竟然也不会把这个渲染错误给修正。除非我直接把这个 state 作为错误 DOM 的 key，强制要求其重新渲染。

我下意识认为是 DOM diff 的问题。

直到我看了 React 的文档。

## 解决方案

用 isMLayout 初始化一个 state，然后再进行分析。
