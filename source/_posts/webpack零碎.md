---
title: webpack零碎
date: 2017-03-09 16:06:44
categories: [框架/库/工具, webpack]
tags: [webpack]
---

## publicPath

迷之总忘！Come on！让我们来记一下！
在生产环境中，静态资源（css、js、图片等）不一定会像开发环境一样，通过相对路径就可以取得。比如html放在一个位置，而js和css被迫放在另外的位置。此时，配置`publicPath`，会将改变对静态文件引入的路径。比如，你的静态文件放在cdn，`publicPath`就可以写为一个完整的地址，那么所有文件都会从这个地址取得。如`publicPath`为`https://www.baidu.com`，则取文件会变为`http://www.baidu.com/main.css`，静态文件取静态文件（比如css取图片）也会从这个路径获取。
将publicPath改为绝对路径同理。
publicPath默认是相对路径`./`，相对于`index.html`。如果部署到生产环境的目录结构与开发时没有任何不同，则不需要设置。

这样，也能看出`publicPath`和`path`的区别了。实际上，除了名字相似，并没有什么可见的联系。`path`是打包时，生成文件“放在哪里”的路径。而`publicPath`是引入打包出的文件时，判断“要从哪里引入”的路径。