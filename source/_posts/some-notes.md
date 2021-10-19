---
title: 零碎笔记
date: 2021-09-09 09:37:20
tags:
---

* useEffect，异步执行。也就是在浏览器渲染全部完成之后，浏览器通知 React 自己处于空闲阶段。此时 React 开始执行 useEffect 产生的函数。
* useLayoutEffect，同步执行。执行时机和 componentDidMount componentDidUpdate 一样。此时，内存中的真实 DOM 已经变化，但还没有渲染在屏幕上。会同步执行，阻塞渲染。但是，如果要对 DOM 进行操作，在这个阶段执行，然后把更改一起渲染在屏幕，可以减少一次重绘和回流。
