---
title: React 和 Redux 中的不可变性：全面指引
date: 2018-10-16 16:41:29
categories: [框架/库/工具, React.js]
tags: [JavaScript, React.js]
---

不可变性（immutability）可以是个令人困惑的话题，一般来说，它会在 React、Redux 和 JavaScript 中到处出现。

你可能碰到过更改了props，但 React 组件不重新渲染的 bug。有人说：“你应该做不可变性的状态更新。”可能你或者你的同事经常写使状态可变的 Redux reducers，且你不得不不断纠正他们（reducers，或者你的同事😄）。

纠正这个很棘手。它真的很微妙，特别是如果你不确定该追求什么。况且说真的，如果你不确定为什么这很重要，那就很难去在意它。

这份导航会解释“不可变性”是什么，以及如果在你自己的应用中写不可变性的代码。

## 不可变性是什么

首先，“不可变”是“可变”的反义词——“可变”意味着可以变化，可能会出问题。

所以某个东西是不可变的，就是说，这个东西不能够被改变。

极端的讲，这意味着比起传统地直接改变值，你应该始终创建新的值去取代旧的值。JavaScript 没有这么极端，但是一些语言完全不允许“可变”（Elixir、Erlang、ML 等等）。

尽管 JavaScript 不是纯粹的函数式语言，但它有时候可以假装是。在 JS 中，某些数组操作是不可变的（就是返回一个新数值，而不是修改原本的）。字符串操作总是不可变的（变化会创建一个新字符串）。并且，你也可以自己写不可变的函数。你只需要意识到一些规则。

### 突变（mutation）的代码示例

我们可以通过一个例子来看可变性是怎样的。比如以下这个 `person` 对象：

```
let person = {
  firstName: 'Bob',
  lastName: 'Loblaw',
  address: {
    street: '123 fake St',
    city: 'Emberton',
    state: 'NJ',
  }
}
```

接下来，我们写一个函数，给这个人赋予超能力：

```
function giveAwesomePowers(person) {
  person.specialPower = 'invisibility'
  return person
}
```

好了，这样，每个人都得到了同样的超能力。不管怎么样，隐身是很棒的！

让我们给 Loblaw 先生赋予超能力：

```
// 刚开始，Bob 没有超能力 :(
console.log(person)

// 接着，我们调用函数
let samePerson = giveAwesomePowers(person)

// 现在，Bob 有超能力了！
console.log(Bob)
console.log(samePerson)

// 尽管如此，在其他方面，他仍旧是同一个人
console.log('Are they the same?', person === samePerson) // true
```

`giveAwesomePowers` 函数改变了传入其中的 `person`。运行这段代码，你能看到，我们第一次打印 `person`，Bob 没有 `specialPower` 属性。然而接下来，第二次，他突然有了隐身的 `specialPower`。

也就是说……

最近太忙，TBC……
