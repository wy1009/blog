---
title: Next.js 本地开发配置 https
date: 2021-12-20 16:25:10
categories: [框架/库/工具, Next.js]
tags: [JavaScript, Next.js, HTTPS]
---

一搜其实不少博客在说，比如 [Using HTTPS on Next.js local development server](https://dev.to/nakib/using-https-on-next-js-local-development-server-bcd) 这篇文章写的就是一个很常用的解法。
为了防止链接失效，见到说下这篇文章的做法。先通过 mkcert 生成 https 证书，将生成的证书拷贝到项目中，然后自己写一个 server.js，引用该证书，且仅在本地开发时启动。

mkcert 使用方法见：[mkcert 文档](https://github.com/FiloSottile/mkcert#mkcert)

重写 server.js 见：[Next.js 自定义 Server](https://www.nextjs.cn/docs/advanced-features/custom-server)

<!-- more -->

落实到代码：

``` JavaScript
// server.js

const { createServer } = require('https')
const { parse } = require('url')
const next = require('next')
const fs = require('fs')

const dev = process.env.NODE_ENV !== 'production'
const app = next({ dev })
const handle = app.getRequestHandler()

const httpsOptions = {
  key: fs.readFileSync('./https-cert/localhost-key.pem'),
  cert: fs.readFileSync('./https-cert/localhost.pem'),
};

app.prepare().then(() => {
  createServer(httpsOptions, (req, res) => {
    // Be sure to pass `true` as the second argument to `url.parse`.
    // This tells it to parse the query portion of the URL.
    const parsedUrl = parse(req.url, true)
    handle(req, res, parsedUrl)
  }).listen(3000, (err) => {
    console.log(err, 'err')
    if (err) throw err
    console.log('> Ready on http://localhost:3000')
  })
})
```

本质就是把官方例子的 createServer 从 http 改为 https，并配置了本地的 ssl 证书。

但是过程中仍旧遇到了很多问题，很久才调通。

## 无法访问到启动的服务

走的就是上面博客的写一个 server.js 的方式。访问 https://localhost:3000 ，Chrome 无法访问，显示 ERR_SSL_KEY_USAGE_INCOMPATIBLE。
于是下意识访问 http://localhost:3000 ，仍旧无法访问。这时候，我就认为是启动的服务有问题。
此时非常迷茫，没有看到任何报错，反复对比代码也没有问题。把启动的 createServer 改为从 http 包再访问就是正常的。所以怀疑是 https 包的问题。根据网上的回复不断尝试重装包、重新启动服务和更改服务端口号到 443，都没有效果。
此时看到一个 github issue，有人提示启动 https 服务需要访问 https 地址，提问者回复确实是没有访问 https，访问的是 http 地址，所以才导致无法访问到服务。
这时候，我才意识到，用 https 包启动的服务，使用 http 方法是无法访问到的。所以服务本身很可能是没有问题的，我只是被 http 无法访问给误导了。
这时候我意识到，我应该解决的是用 https 访问时看到的报错。

## 使用 https 访问服务报错

感觉是证书安装有问题，在钥匙串里查了半天，也 uninstall 然后重新 install 生成了新的证书。但是仍旧没有效果。

最终在这个 [issue](https://github.com/FiloSottile/mkcert/issues/253) 中查到了原因。该 issue 中有一个回复：

> If you are seeing this error, it means you are trying to use the root CA directly. You are NOT supposed to use rootCA.pem directly. Instead, generate a certificate for localhost or example.com with mkcert localhost or mkcert example.com and use that.

我确实直接使用了根证书。因为在按照官方文档使用 `mkcert example.com "*.example.com" example.test localhost 127.0.0.1 ::1` 命令时，实际上应该创建出新的证书。而不知为何我并没有成功创建。
所以，当我使用 `mkcert -CAROOT` 查看证书文件时，我的目录下就只有两个根证书，我就认为应该使用根证书。
上面的 issue 解决了我的问题。不能直接使用根证书，而应该根据不同的域名生成不同的证书，然后引入进去。

Problem Solved.
