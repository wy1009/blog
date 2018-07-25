---
title: Nuxt.js使用笔记
date: 2018-01-09 17:19:31
categories: [框架/库/工具, Vue.js]
tags: [JavaScript, Nuxt.js]
---

## Nuxt.js原理

一开始不知道这是怎么做到服务器端渲染的，后来写着写着就明白了。启动一个Server，执行前端代码。我司后端用的是nginx，配置是将请求直接转发到这个Server。
初次请求页面时，前端代码由Server端执行，生成一个内容完整而不是依赖js动态执行展示内容的HTML文件，再返回给Client端。
之后则正常走单页面应用的逻辑。

## 与浏览器执行代码的不同之处

初次请求由Server端执行也带来了一些问题。在初次请求时，组件created之前的步骤全部由Server端执行，包括middleware、asyncData、beforeCreate。

### cookie

因为是在Server端执行代码，本地将没有Client端有的cookie，因此：

- 在发送ajax请求时，需要通过req.headers.cookie将cookie取出，然后设置axios，以此做到发送ajax时携带cookie。为axios设置的cookie不能为undefined，否则报错“”value” required in setHeader(“cookie”, value)”；
- 在收到响应后，如果后台set cookie，在Server端也无法正常达到将cookie set到Client端上的效果，而需要利用res.setHeader(‘set-cookie’, ‘cookie=cookie’)，从Server端设置Client端的cookie。

### 没有window/document

1. 涉及到跳转，用context中的redirect代替window.location.href。需要注意，该方法在Client端执行时也可以使用，但是，在客户端执行时跳转的是相对路径。即，在客户端执行`redirect('https://www.google.com')`，会跳转到`http://hostname/https://www.google.com`。因此，需要做好判断，在Client端用location.href代替redirect；
2. 涉及到cookie，参考上一节。

### 关于store

Server端和Client端store内的值可以互通，即在中间件等Server执行代码时，对store做出的更改，在Client端仍可以取到。
但是，如果在Server端执行redirect（或在Client端执行location.href），则不再能取到store值，即使redirect的目标路径正是本单页面应用的路径。因为在Server端redirect，相当于重新开启一个单页面应用。
但是，如果在Client端执行redirect到本单页面应用的路径，则store保存。因为该行为为应用内跳转，单页面应用并没有被关闭。
