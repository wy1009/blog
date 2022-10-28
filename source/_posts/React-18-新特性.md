---
title: React 18 新特性
date: 2022-10-27 12:51:12
categories: [框架/库/工具, React.js]
tags: [JavaScript, React.js]
---

能够看出来，React 最近整体的升级思路都是试图从框架的层面去优化业务代码，优化用户体验。
16.8 是对渲染之外的部分动手，可以打算渲染之前的逻辑做到及时响应用户。Lin Clark 在介绍 Fiber 渲染器的时候还提到过，渲染是不会打断的，否则会造成 UI 出错。
而 React 18，核心其实就是，开始对渲染动手了。

# React 17

## 支持渐进式升级

可以将大部分功能升级到 React 18，但保留部分子路由或懒加载对话框在 React 17。
但当然推荐还是一次性升级。

## 事件委托

![React 17 事件委托挂载节点](https://wx1.sinaimg.cn/mw2000/7b1152ffly1h7jr9v1w8rj22p4234x6r.jpg)

## 一些破坏性更改

### 对标浏览器

* onScroll 事件不再冒泡，以防止出现常见的混淆。
* React 的 onFocus 和 onBlur 事件已在底层切换为原生的 focusin 和 focusout 事件。它们更接近 React 现有行为，有时还会提供额外的信息。
* 捕获事件（例如，onClickCapture）现在使用的是实际浏览器中的捕获监听器。

这些更改会使 React 与浏览器行为更接近，并提高了互操作性。

### 去除事件池

``` JavaScript
function handleChange(e) {
  setData(data => ({
    ...data,
    // This crashes in React 16 and earlier:
    text: e.target.value
  }));
}
```

因为 React 在事件处理函数结束后就把事件清空放回到事件池里去了，导致这个问题。需要调用 `e.persist` 解决。

### 副作用清理时间

副作用清理方法异步执行。
需要同步执行，用 `useLayoutEffect`。
React 17 将在运行任何新副作用之前执行所有副作用的清理函数（针对所有组件）。React 16 只对组件内的 effect 保证这种顺序。

### 返回 undefined 报错

forwardRef 和 memo 组件的行为会与常规函数组件和 class 组件保持一致。在返回 undefined 时会报错

### 原生组件栈

17 之前，控制台打印堆栈，无法做到像原生 JS 一样跳转到对应位置。17 可以做到这一点。

### 移除私有导出

不再将 React 内部组件暴露给其他项目。

## 全新的 JSX 转换

与 Babel 合作，可以单独使用 JSX 而无需引入 React。

以前，会将源代码：

``` JavaScript
import React from 'react';

function App() {
  return <h1>Hello World</h1>;
}
```

转换为：

``` JavaScript
import React from 'react';

function App() {
  return React.createElement('h1', null, 'Hello world');
}
```

17 开始，会转换为：

``` JavaScript
// 由编译器引入（禁止自己引入！）
import {jsx as _jsx} from 'react/jsx-runtime';

function App() {
  return _jsx('h1', { children: 'Hello world' });
}
```

# React 18

React 18 引入并发特性。
* React 18 之前，同步渲染：一旦更新开始渲染了，在用户可以在屏幕上看到结果之前，没有任何任务可以中断它。
* React 18，并发渲染：渲染也可以被打断。先评估整棵树再执行 DOM 变化，React 保证 UI 的一致性。

这样，渲染也不阻塞主线程。
另外，React 还可以在改变状态的时候删除 UI，然后再状态又变回来的时候把这部分 UI 重新添加回去。日后还打算添加一个 Offscreen 组件，在后台准备新 UI。

## 自动批处理

React 18 之前，自动批处理只生效于事件监听、生命周期方法等在 react 生命周期之内的方法。如果在promise 等回调方法里，则不会批处理。
现在都可以批处理了。
可以选择不批处理。利用 flushSync。一般应该用不上，但你要是真利用了同步更新的特性，那就可以用这个 api。

## Suspense on Server

以前，SSR 是一个全有或者全无的事，一个组件可以影响整个页面的性能。
而服务端 Suspense 可以：

* 一部分慢不会影响到整个页面
* 更早显示 HTML，剩下的部分通过流下发
* Code Splitting 完全融入服务端渲染

详细笔记在备忘录里，不太好弄出来。大家要么直接看 [Streaming Server Rendering with Suspense](https://reactjs.org/blog/2021/12/17/react-conf-2021-recap.html#streaming-server-rendering-with-suspense)。分享的时候讲的相关内容其实都在这里。

## 新的 API

参考 [官网的 React 18 新 API](https://zh-hans.reactjs.org/blog/2022/03/29/react-v18.html#new-feature-automatic-batching)。主要就是 `useTransition` 和 `useDeferredValue`，主要都是为了优化渲染，手动区分渲染的优先级。

# 参考

* [event.persist() should be called when using React synthetic events inside an asynchronous callback function](https://deepscan.io/docs/rules/react-missing-event-persist)
* [React 17 破坏性变更](https://zh-hans.reactjs.org/blog/2020/08/10/react-v17-rc.html#other-breaking-changes)
* [React 17 JSX 转换](https://zh-hans.reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html)
