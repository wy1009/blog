---
title: Next 学习过程中想知道的问题
date: 2021-12-07 20:17:26
categories: [框架/库/工具, Next.js]
tags: [JavaScript, Next.js, 原理]
---

先做个笔记。等需求完成之后再细看。

## 如何做到从预渲染页面到可交互页面的？

> 每个生成 HTML 与该页所需的最小 JavaScript 代码相关联。当一个页面被浏览器加载时，它的 JavaScript 代码就会运行并使页面完全交互。(此过程称为hydration.)

这个是如何做到的呢？在编译过程中就把所有可交互内容（如绑定事件 onClick 等）收集起来，JS 执行之后组装到预渲染的 html 内容上？

需要看看原理。

## getStaticProps

在预渲染阶段是请求到数据了。在实际客户端运行的过程中呢？就不重复取这个数据了吗？
