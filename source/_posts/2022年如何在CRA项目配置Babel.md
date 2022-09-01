---
title: 2022 年，如何在 CRA 的项目中做浏览器兼容
date: 2022-07-12 18:36:36
categories: [框架/库/工具, Babel]
tags: [Babel, CRA]
---

# CRA 项目如何兼容低版本浏览器

对于兼容低版本浏览器的需求，[CRA 官方文档](https://create-react-app.dev/docs/supported-browsers-features)写得很清楚。只要引入 `react-app-polyfill` 就可以解决绝大部分情况的问题。
但实际使用时，我们发现，至少在今天（2022.7.12），对于超低版本浏览器（安卓 5.1.1，Chrome 42），`react-app-polyfill` 并没有提供对实例方法 `arr.includes`，以及浏览器 API `URLSearchParams` 的支持。因此，不得已自己使用 babel 作出支持。

# CRA 项目引入 babel

原本的 `@babel/polyfill` 已经在 babel 7.4.0 版本中被弃用，如今完整的 polyfill 已经转移到了 `core-js` 中。实际上，直接 `import core-js` 就可以解决所有问题了。但这就意味着引入了所有的 polyfill，不管你想支持的浏览器版本，不管你实际使用了哪些方法。这势必会造成包的臃肿，因而需要对 babel 进行配置。
贴心的 babel 提供了两种按需加载方式，`@babel/preset-env` 的 `useBuiltIns` 可配置为 `entry` 或 `usage`。
如果配置为 `entry`，babel 则会通过对 browserlists 的配置，按需引入 `core-js` 的代码，如官方示例：

你的代码：

``` JavaScript
import "core-js";
```

转换后：

``` JavaScript
import "core-js/modules/es.string.pad-start";
import "core-js/modules/es.string.pad-end";
```

而 `usage` 参数，则代表着更精确的按需加载。**无需**在文件开头 `import 'core-js'`，babel 在处理你的代码的时候，会直接解析你的代码实际使用了哪些语法，然后有选择性地引入。

该参数是在 babel 配置文件中配置的：

``` JSON
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "useBuiltIns": "usage",
        "corejs": "3.23"
      }
    ]
  ]
}
```

然而，现实是，我花了不少时间尝试，分别在 `package.json` 和 `babel.config.json` 中写了配置，似乎都并不生效。无论怎么改配置，打出来的 js 文件夹始终是 1.2M，并没有按需加载的效果。这让我感到很迷茫。

于是查了下[CRA 源码](https://github.com/facebook/create-react-app/blob/3880ba6cfd98d9f2843217fd9061e385274b452f/packages/react-scripts/config/webpack.config.js#L411)，发现不仅是 webpack 配置，CRA 创建的项目默认连 babel 配置都不支持。那就只能用 `react-app-rewired` + `customize-cra` 来改了~看文档就会用，很方便很直观。

值得一提的是，如果 `customize-cra` 没有提供对应的方法，可以用以下方法去补充：

``` JavaScript
const {
  override,
  useBabelRc,
  addWebpackPlugin,
} = require("customize-cra")
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin

// 自定义方法
const setConcatenateModules = cm => (
  config => {
    config.optimization.concatenateModules = cm
    return config
  }
)

module.exports = override(
  useBabelRc(),
  setConcatenateModules(false),
  addWebpackPlugin(new BundleAnalyzerPlugin()),
)
```

即自己写一个方法来变更配置。

至此，我终于成功配置了 babel。

# 成功配置 babel 后

在配置完 babel 后，打包推到测试服，5.1.1 的手机仍旧白屏。debug 发现，是 `URLSearchParams` 仍旧不支持。这个很好理解，`useBuildIns: usage` 是在我们使用某方法后才会将对应 polyfill 引入的，而 `URLSearchParams` 实际上是 `react-router-dom@6` 自行使用的。而在 CRA 项目的 webpack 配置中，`babel-loader` 只会处理 `src` 路径下的文件，当然不包括 `node_modules`。

因此，在文件开头手动引入对应 polyfill `import 'core-js/web/url-search-params'`。

至此，项目的兼容性问题就完全解决了。

# 一个未解之谜

我的项目中有多个打包命令，用于区分是否生成 sourcemap，请求的接口是测试环境还是正式环境等。
`yarn build:prod` 打包，请求的接口是正式环境，而 `yarn build` 请求的是测试环境。
此前，在 `yarn build:prod` 的情况下，配置已经没有什么问题了。但是改成 `yarn build` 打算提测后，控制台忽然就出现了 `Object.assign is not a function.` 的报错，且页面白屏。
考虑到两个环境的差异就只有请求接口的不同，我第一反应就是接口环境不同导致重定向的页面不同，造成访问的页面不同。所以正式环境没有触发 bug，测试环境触发了。
以这个思路查了半天，发现并不是这个问题。甚至发现客户端其实会拦截所有前端请求更改为正式环境（历史 Charles 抓包也显示确实从未请求过测试环境），所以两个包的运行情况应该是一模一样的才对。
虽然不知道两个运行情况相同的包为什么会有不同的表现，但我还是打算先猜测解决方案。首先从 browserlists 入手，利用[这个地址](https://liulanmi.com/labs/core.html)，我查到，我手头这个 5.1.1 的机器，webview 内核版本为 Chrome 42，而我的配置的浏览器支持似乎高于这个版本：

``` Shell
# @babel/preset-env debug 模式下的控制台输出
Using targets:
{
  "android": "4.4.3",
  "chrome": "79",
  "edge": "97",
  "firefox": "96",
  "ie": "11",
  "ios": "12.2",
  "opera": "82",
  "safari": "13.1",
  "samsung": "16"
}
```

虽然支持的安卓版本低于 5.1.1，但是 chrome 版本是高于 42 的。我怀疑是这个原因导致的，就在 browserlists 中加入了 `chrome 30`。

然后问题就解决了。

但这还是不能解释一个问题，就是为什么正式环境的包就没有同样的问题。因此，我将 `chrome 30` 的配置移除了，想再次观察情况。然后自此开始，`Object.assign` 这个 bug 就再也没有出现了。

我怀疑是缓存的问题，删除了 `node_modules/.cache`，不复现。我干脆直接让其他同事拉代码部署了一下，同样不再复现。

而因为发现 `debug` 的用处太晚，我甚至没有及时地监控到之前问题复现的时候，`object.assign` 的 polyfill 是否被正确引入了。这件事就暂时成为了未解之谜…………

总之还是记录一下，debug 真的非常好用，不仅可以告诉你 target 的系统/浏览器内核版本，还会在 `usage` 的时候告诉你因什么文件而引入了什么 polyfill，非常非常好用了。

``` Shell
# @babel/preset-env debug 模式下的控制台输出
[/Users/wangyi/Documents/edg/zhixue/src/bridge/index.ts]
The corejs3 polyfill added the following polyfills:
  es.regexp.exec { "android":"4.4.3", "chrome":"79", "edge":"97", "firefox":"96", "ie":"11", "ios":"12.2", "opera":"82", "safari":"13.1", "samsung":"16" }
  es.regexp.test { "android":"4.4.3", "chrome":"79", "edge":"97", "firefox":"96", "ie":"11", "ios":"12.2", "opera":"82", "safari":"13.1", "samsung":"16" }
  es.object.assign { "android":"4.4.3", "chrome":"79", "edge":"97", "firefox":"96", "ie":"11", "ios":"12.2", "opera":"82", "safari":"13.1", "samsung":"16" }
```

# Vue 项目的一个兼容问题

最初白屏，发现是箭头函数导致。项目中所有代码应该都过了 babel，认为有可能是 browserlist 覆盖的版本过高，或者是 node_modules 中的内容带出来的。
查看打包内容，其中有一个 =>，认为是 node_modules 带来的。
查看 vue-cli 文档，发现 transpileDependencies 可以显式设置将什么包过一遍 babel-loader。但问题是，我们无法定位我们打包文件中的那个箭头所对应的包。
此时，忽然发现物理机打包没有同样的问题，ci 构建却有问题。
这其中理论上讲不应该有什么区别。
考虑到是第三方的包有问题，包的内容本身并没有经过 babel-loader，只能理解为是包本身的内容有差异。将两边打包后的文件拉出来，发现 ci 构建的那个 build 文件确实是不同的，里面有大量的箭头函数。所以我们本地构建包看到的那个箭头函数并没有引发问题，实际引发问题的是 ci 构建得到的大量箭头函数。
如果是这样，可能是包的版本问题。查看 ci 构建命令，发现确实 ignore 了 lockfile（打包命令含 --no-lockfile）。询问运维，这（可能是为了解决内部项目构建速度的问题，统一将拉包的 registry 改为内部的一个源）。
