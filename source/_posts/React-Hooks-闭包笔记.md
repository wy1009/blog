---
title: React Hooks 闭包笔记
date: 2022-10-25 21:14:01
tags:
---

每次都会忘记细节，记个笔记。

``` JavaScript
// test 在作用域链上端，形成闭包
const [test, setTest] = useState(0)

// test 仍旧会形成闭包，因为 test 虽然是一个引用值，但应该在每次渲染时都重新创建，其引用地址发生改变
const test = useState(0)

// test 为一个引用值，且注意在 setTest 的时候不改变其引用，则不受闭包影响。因为 state 被记录在 fiber 上，每次创建的都是同一个值，引用值始终没变。
const [test, setTest] = useState({})
```
