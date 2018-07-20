---
title: Vue.js双向数据绑定原理
date: 2017-10-10 15:15:25
categories: [框架/库/工具, Vue.js]
tags: [JavaScript, Vue.js]
---

写个（只有）自己能看懂的流程。

## mvvm.js

1. 数据代理；
2. initComputed；
3. observe(this.data)。

## observer.js

1. 监听每一个值的get/set，每一个值跟随一个dep。如果被取值，将Dep全局保存的watcher推入dep中的watcher列表。

## mvvm.js

1. compile(options.el)。

## compile.js

1. 遍历el中的所有节点，分别处理每个节点。
2. 以text节点为例，分析出是否和data的属性关联，如果关联，新建一个watcher，watcher中包含了此节点、关联的data属性名以及updateFn（一个用于更新节点的函数，比如文本节点，就是node.textContent = val）。

## watcher.js

1. 将与节点相关联的属性名分解开来（a.b.c），挨个取值；
2. 因取值而触发observer.js中的get方法，将该watcher推入该属性的dep中；
3. watcher中记录下该watcher推入过哪个dep（dep标id），避免重复推入（update时也需要get值，此时不需要重复推入）。
