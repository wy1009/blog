---
title: 怎么会有人每次都会忘记 hooks 闭包的细节啊
date: 2022-09-01 20:37:37
categories: [框架/库/工具, React.js]
tags: [JavaScript, React.js, 源码]
---

大概的原理总是很清楚：因为方法一直是以前某次 render 调用的那一个，所以引用的值也一直是旧的。
但是为什么呢？state 值不是在 fiber 里吗？fiber 肯定不会是旧的呀。
每次想清楚都会忘记，干脆记一下。
首先，state 的值确实是从 fiber 取的，过时的值是实际是 Component Function 作用域的值，也就是作用域链上每次加的那个变量对象。
所以，之前的那个想法也是没错的：如果 state 是一个引用类型的值，那就不会存在这个闭包问题。因为每次 state 都是从 fiber 取的，如果 state 是引用类型，地址从未改变过，那么哪怕是旧的值，只要以不改变引用的方式重新 set 了，也会后续 render 也会读到最新值——因为引用不变。

![引用类型代码](https://wx3.sinaimg.cn/mw2000/7b1152ffly1h5re6lw0gmj20mq0betbp.jpg)
![如果以不改变引用的形式改变 state，不会有闭包导致的问题](https://wx4.sinaimg.cn/mw2000/7b1152ffly1h5re6r83dzj206a08s75e.jpg)

控制台输出两次是因为有两个组件在跑，不必在意。
