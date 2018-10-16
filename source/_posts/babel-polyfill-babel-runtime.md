---
title: babel-polyfill & babel-runtime
date: 2018-03-28 15:52:57
categories: [框架/库/工具, webpack]
tags: [JavaScript, ES6, babel, webpack]
---

之前明明知道，不知道为什么又变得不确定……“好记性不如烂笔头”系列。
为了验证一下，又找不到火狐旧版本入口，还专门去了类似“太平洋下载站”的地方下了个旧版的火狐哈哈哈哈哈哈哈有十年多没进过这类网站了吧。

## .babelrc

```
{
  "present": [
    "es2015",
    "react",
    "stage-2"
  ],
  "plugins": []
}
```

Present是按照ECMAScript草案来组织的，通常可以分为以下三大类：

1. 已经被写入ECMAScript标准中的特性，由于之前每年都有新特性被加入到标准中，又可以细分为：
  - es2015包含在2015里加入的新特性；
  - es2016包含在2016里加入的新特性；
  - es2017包含在2017里加入的新特性；
  - env包含当前所有ECMAScript标准里的最新特性。

env包含了es2015、es2016和es2017。

2. 被社区提出，但还没有写入ECMAScript标准里的特性，这其中又分为以下四种：
  - stage0 只是一个美好激进的想法，有 Babel 插件实现了对这些特性的支持，但是不确定是否会被定为标准；
  - stage1 值得被纳入标准的特性；
  - stage2 该特性规范已经被起草，将会被纳入标准里；
  - stage3 该特性规范已经定稿，各大浏览器厂商和 Node.js 社区开始着手实现。

stage0包含stage1，stage1包含stage2，stage2包含stage3。

3. 为了支持一些特定应用场景下的语法，和ECMAScript标准没有关系。例如`babel-preset-react`是为了支持React开发中的JSX语法。

<!-- more -->

## babel-polyfill

babel编译为了保证正确的语义，只能转换语法，而不是增加或修改原有的对象和属性。所以babel不会转换新的API，例如`Iterator`、`Generator`、`Set`、`Map`、`Proxy`、`Reflect`、`Symbol`、`Promise`等全局对象，以及一些定义在全局对象上的方法，如`Object.assign`。如果想使用这些新的对象和方法，必须使用`babel-polyfill`。

```
npm install --save babel-polyfill
import 'babel-polyfill'

// 或者
require('babel-polyfill')

// 或者

module.exports = {
  entry: ['babel-polyfill', './app/js']
}
```

## babel-runtime

babel转译后的代码，要实现与原代码同样的功能，需要借助一些帮助函数。帮助函数可能会重复地出现在一些模块里，导致编译后的代码体积过大。babel为解决这个问题，提供了单独的包`babel-runtime`供编译模块复用工具函数。

启用插件`babel-plugin-transform-runtime`之后，babel就会使用`babel-runtime`中的工具函数。除此之外，babel还为原代码的非实例方法（如`Object.assign`，实例方法指的是`&#39;foobar&#39;.includes(&#39;foo&#39;)`）和`babel-runtime/helps`下的工具函数自动引用了`polyfill`，这样可以避免污染全局命名空间。

具体项目还是需要用到`babel-polyfill`，只使用`babel-runtime`的话，实例方法无法工作。但JavaScript库和工具可以只是用`babel-runtime`，在实际项目中使用这些工具，项目本身会提供`babel-polyfill`。

## 参考文献

- [babel的polyfill和runtime的区别](https://segmentfault.com/q/1010000005596587)
- [深入浅出webpack](http://webpack.wuhaolin.cn/3%E5%AE%9E%E6%88%98/3-1%E4%BD%BF%E7%94%A8ES6%E8%AF%AD%E8%A8%80.html)
