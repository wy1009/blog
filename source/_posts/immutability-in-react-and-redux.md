---
title: React 和 Redux 中的不可变性：全面指引
date: 2018-10-16 16:41:29
categories: [框架/库/工具, React.js]
tags: [JavaScript, React.js]
---

不可变性（immutability）可能是个令人困惑的话题，它总会在 React、Redux 和 JavaScript 中到处出现。

你可能碰到过已经更改了 props，但 React 组件不重新渲染的 bug。有人说：“你应该做不可改变状态（immutable state）的更新。”可能你或者你的同事经常写改变状态（mutate state）的 Redux reducers，且你不得不不断纠正这些（reducers，或者你的同事😄）。

纠正这个很棘手。它真的很微妙，特别是如果你不确定为什么要纠正。况且说真的，如果你不确定为什么这很重要，那就很难去在意它。

这份导航会解释“不可变性”是什么，以及如果在你自己的应用中写不可变性的代码。

## 不可变性是什么

首先，“不可变”是“可变”的反义词——“可变”意味着可以变化，可能会出问题。

所以某个东西是不可变的，就是说，这个东西不能够被改变。

极端的讲，这意味着比起传统地直接改变值，你应该始终创建新的值去取代旧的值。JavaScript 没有这么极端，但是一些语言完全不允许“可变”（Elixir、Erlang、ML 等等）。

尽管 JavaScript 不是纯粹的函数式语言，但它有时候可以假装是。在 JS 中，某些数组操作是不可变的（就是返回一个新数值，而不是修改原本的）。字符串操作总是不可变的（变化会创建一个新字符串）。并且，你也可以自己写不可变的函数。你只需要意识到一些规则。

<!-- more -->

## 改变（mutation）的代码示例

我们可以通过一个例子来看可变性是怎样的。比如以下这个 `person` 对象：

``` JavaScript
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

``` JavaScript
function giveAwesomePowers(person) {
  person.specialPower = 'invisibility'
  return person
}
```

好了，这样，每个人都得到了同样的超能力。不管怎么样，隐身是很棒的！

让我们给 Loblaw 先生赋予超能力：

``` JavaScript
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

也就是说，在这个函数改变了传入的 `person` 之后，我们就再也不知道以前的 `person` 是什么样子了。它被永远地改变了。

`giveAwesomePowers` 返回的对象和传入 `giveAwesomePowers` 的对象是同一个，但是这个对象里面已经被搞乱了，它的属性已经被改变了。它已经被改变（be mutated）了。

因为很重要，我想再说一次：这个对象里面已经被改变了，但是这个对象的引用没有变。它和外面的对象是同一个对象（所以检查是否相等的 `person === samePerson` 的结果是 `true`）。

如果我们不想让 `giveAwesomePowers` 函数改变 `person`，我们需要做一些改动。不管怎么样，我们先看看是什么让一个函数变得纯粹（pure），因为它和不可变性密切相关。

## 不可变性的规则

要使一个函数纯粹，需要遵循以下规则：

1. 输入相同的值，纯函数会永远返回相同的值；
2. 纯函数不产生任何副作用。

## 什么是“副作用”

“副作用”是一个意思很广的术语，但基本上，它意味着修改了直接函数的范围之外的东西。举一些例子：

* 改变传入的参数，像 `giveAwesomePowers` 一样；
* 改变任何函数外面的值，比如全局变量，或者 `document`/`window` 上的值；
* 调用 API；
* `console.log()`
* `Math.random()`

调用 API 这条可能让人比较意外。毕竟，调用像是 `fetch('/users')` 好像根本不会改变 UI。但是，如果你调用了 `fetch('/users')`，它真的不会改变任何地方吗？比如 UI 之外呢？

实际上，它会在浏览器的网络日志中创建一条日志。它会创建（且过会可能会关闭）一个与服务器的网络连接。且一旦连上了服务器，服务器可以做任何它想做的事，包括唤起其他服务做更多改变。最少最少，它也会在日志文件中记一条日志（这也是一个改变）。

所以说，“副作用”是一个含义很广的属于。

下面是一个没有副作用的函数：

``` JavaScript
function add(a, b) {
  return a + b
}
```

你可以调用它一次，也可以调用一百万次，但世界上任何其他的东西都不会因此而改变。这符合规则 2：没有副作用。

另外，每次你调用这个函数，比如 `add(1, 2)`，你都会得到同样的答案。不管你调用多少次 `add(1, 2)`，都会得到同样的答案。这符合规则 1：同样的输入得到同样的输出。

## 会改变的 JS 数组方法

某些数组方法会改变数组：

* push（在尾部添加一项）
* pop（从尾部移除一项）
* shift（从头部移除一项）
* unshift（在头部添加一项）
* sort
* reverse
* splice

是的，JS 数组的 `sort` 方法不是不可变的！它会原地重排数组。

如果你需要做这些操作，最简单的方法就是先复制出一个数组，然后在复制出的数组上做操作。你可以用以下方法复制一个数组：

``` JavaScript
let a = [1, 2, 3]
let copy1 = [...a]
let copy2 = a.slice()
let copy3 = a.concat()
```

所以，如果你想在数组上做一个不可变的排序，你可以这样做：

``` JavaScript
let sortedArray = [...originalArray].sort(compareFunction)
```

另外，有一个小拓展（之前坑了我一次），`compareFunction` 需要返回 0、1 或者 -1，而不是布尔值。

## 纯函数只能调用纯函数

一个可能造成问题的做法是，在纯函数中调用非纯函数。

纯粹性要么有，要么没有。如果你写了一个完美的纯函数，但是在结尾调用了其他函数，而这个被调用的函数调用了 `setState`、`dispatch` 或者引起了其他副作用，那你就前功尽弃了。

现在，有些副作用是“可接受的”。用 `console.log` 记录日志就没关系。是的，技术上讲，这是一个副作用，但是它不会影响任何东西。

## 纯函数版本的 `giveAwesomePowers`

现在，我们可以用我们心里的规则重写这个方法了：

``` JavaScript
function giveAwesomePowers(person) {
  let newPerson = Object.assign({}, person, {
    specialPower: 'invisibility',
  })

  return newPerson
}
```

现在有一点不同了。我没没有改变 `person`，而是创建了一个全新的 `person`。

如果你没见过 `Object.assign`，它的作用是把一个对象的属性复制到另一个对象上。你可以向其中传入一系列的对象，它会将它们都合并到一起，从左到右，重复的属性会被重写。（从左到右，指的是执行 `Object.assign(result, a, b, c)`，会把 `a` 复制到 `result` 上，然后是 `b`，然后是 `c`）。

但它做的不是深合成。它只会直接原样复制每个参数的子属性，不会为属性创建副本。

另一种借用 `...` 运算符的写法是：

``` JavaScript
function giveAwesomePowers(person) {
  let newPerson = {
    ...person,
    specialPower: 'invisibility',
  }

  return newPerson
}
```

## 纯函数返回全新的对象

现在，我们可以用我们纯函数版本的 `giveAwesomePowers` 重新跑一遍我们之前的代码了。

``` JavaScript
// 刚开始，Bob 没有超能力 :(
console.log(person)

// 接着，我们调用函数
let samePerson = giveAwesomePowers(person)

// 现在，Bob 的克隆体有超能力了！
console.log(Bob)
console.log(samePerson)

// newPerson 是一个克隆体
console.log('Are they the same?', person === samePerson) // false
```

这与之前有很大的不同，`person` 没有被改变。Bob 没有变。这个函数创造了一个 Bob 的克隆体，有全部相同的属性，加上了能够隐身的能力。

这是函数式编程比较奇怪的地方。对象不断被创建和销毁。我们没有改变 Bob，我们创建了一个 Bob 的克隆体，更改这个克隆体，然后用这个克隆体替换了 Bob。确实有点恐怖。如果你看过一部叫《The Prestige》的电影，这个和它有点像。

## React 更喜欢不可变性

在 React 的案例中，永远不要直接改变 state 和 props 是很重要的，不管组件是函数还是类都是这样。如果你要写类似 `this.state.something = ...` 或 `this.props.something = ...` 的代码，退一步然后试着想想更好的方法。

如果要改变 state，通常使用 `this.setState`。更多的信息可以阅读[《为什么不能直接改变 state》](https://daveceddia.com/why-not-modify-react-state-directly/)。

至于 props，他们是单向的。props 被传入一个组件，但不存在双向通路，至少不能做类似把一个 prop 设为新值的可变的操作。

如果你需要把一些数据传递回父组件，或者在父组件引发一些操作，你可以将一个方法作为 prop 传入子组件，然后在需要与父组件交流的时候，在子组件中调用这个方法。这是一个小例子：

``` JavaScript
function Child(props) {
  // 当按钮被点击时，父组件传入的方法会被调用
  return (
    <button onClick={props.printMessage}>Click Me!</button>
  )
}

function Parent() {
  function printMessage() {
    console.log('you click the button')
  }

  // 父组件传递一个函数给子组件作为 prop
  return (
    <Child onClick={printMessage} />
  )
}
```

## 不可变性对于 `PureComponents` 而言很重要

默认来说，React 组件（不管是函数组件还是类组件，只要继承于 `React.Component` 都会）会在你调用 `setState`，或者父组件重新渲染的时候重新渲染。

一个简单的优化 React 组件性能的方法是，使其成为类组件，然后让其继承于 `React.PureComponent`，而不是 `React.Component`。这样，这个组件将只在它的 state 或者 props 改变的时候重新渲染，而不再会无脑地在每次它的父组件重新渲染的时候重新渲染。

这就是不可变性的来源：如果你要向一个 `PureComponent` 中传递 props，你必须得确定这些 props 都是以不可变的方式更新的。也就是说，如果 props 是对象或者数组，你需要用一个新的（更改过的）对象或数组去替换掉原来的。就像 Bob，杀死他，然后用一个克隆体替换他。

如果你更改了一个对象或者数组的内部——比如更改一个属性，或者推入了新的一项，或者更改了数组中的一项——那么这个对象或数组在引用上仍旧等于旧的自己，所以一个 `PureComponent` 不会注意到它已经改变了，也就不会重新渲染。奇怪的渲染 bug 也就出现了。

记得我们第一个 Bob 和 `giveAwesomePowers` 函数的例子吗？函数返回的对象完全等于传入的那个。这是因为两个变量都引用了同一个对象，只是内部被改变了。

## 引用相等在 JavaScript 中是如何运行的？

“引用相等（Referential Equality）”指的是什么？理解这个很重要。

JavaScript 对象和数组是被储存在内存中的。让我们假设内存里的位置就像盒子。变量名指向盒子，而盒子保存着实际的值。

![内存”盒子“](/images/memory-box-1.png)

在 JavaScript 中，这些盒子（内存地址）是未命名且不可知的。你无法看到一个变量实际指向的内存地址。（在一些其他语言中，比如 C，你可以实际地检查变量的内存地址并查看它的位置。）

如果你重新给变量赋值，变量会指向一个新的内存位置。

![内存”盒子“](/images/memory-box-2.png)

如果你改变了这个变量的内部，变量仍会指向原本的地址。

![内存”盒子“](/images/memory-box-3.png)

就像是重新装修房子，哪怕装上了新的墙、厨房、卧室或者游泳池，这个房子的地址仍旧是不变的。你没必要提醒亲戚往哪里寄礼物，因为你仍就住在同样的地方。

**这是重点：**当你用 `===` 操作符比较两个对象或者数组时，JavaScript 实际上比较的是他们指向的地址，也就是它们的引用。JS 不会看这个对象的内部。这也就是”引用相等“的意思。

所以，如果你改变一个对象，你改变的是这个对象的内容，而不会改变它的引用。

另外，如果你把一个对象赋值为另一个对象（或者把它作为一个函数的参数传入，这做的实际上是同一件事），另一个对象只是指向了同样的内存位置，和第一个对象一样。就像巫毒娃娃一样，你对第二个对象做的操作也会直接影响到第一个对象的值。

具体参见下面的代码：

``` JavaScript
// 创建一个变量 `crayon`，指向一个盒子（未命名），盒子中保存着对象 `{ color: 'red' }`
let crayon = { color: 'red' }

// 改变 `crayon` 的一个属性不会改变它指向的是哪个盒子
crayon.color = 'blue'

// 把一个变量赋值为另一个对象或者数组，只是把新变量的指向指到了同样的一个盒子上
let crayon2 = crayon
console.log(crayon2 === crayon)

// 现在，任何对 `crayon2` 的更改也会影响 `crayon`
crayon2.color = 'green'
console.log(crayon.color) // 改为 green
console.log(crayon2.color) // 也变成了绿色

// 因为这两个变量在内存中指向同一个对象
console.log(crayon2 === crayon) // true
```

## 为什么不深入比较是否相等呢？

在声明两个对象相等之前，先检查它们的内部看起来更加“正确”。虽然确实是这样，但是太慢了。

有多慢呢？这取决于被比较的对象。有一万个子属性和孙属性的对象肯定比只有两个属性的对象要慢。这是不可预测的。

引用相等的检查被称为“恒定时间”。恒定时间，也就是 O(1)，意味着这个操作永远消耗相同的时间，不管输入的有多大。

而深入检查是否相等则更像是“线性时间”，也就是 O(N)，意味着消耗的时间和对象的属性数量是成比例的。线性时间一般来讲是慢于恒定时间的。

可以这样想，假装 JS 每次对比两个值，比如 `a === b` 都要花上一整秒。现在，你是想要只比较一次，只比较引用，还是想要深入检查两个对象，比较每个属性呢？第二种听起来很慢吧？

现实中，对比一次是否相等要比一整秒快得多得多，但是，“尽可能做最少的工作”的原则仍旧适用。其他条件相同时，使用性能最高的选项，可以让你少花点时间寻找你的 app 为什么这么慢。如果你足够小心（且比较幸运），可能根本就不会慢。

## `const` 能阻止改变吗？

简单说：不会。不管是 `let` 还是 `const` 或者 `var` 都不会阻止你改变一个对象的内部。这三种声明变量的方式都允许你更改其内部。

“但是它被称为‘常量（`const`）’！它应该一直不变呀！”

一般来说是这样。但 `const` 只会阻止你重新分配引用，而不会阻止你改变这个对象。这是一个例子：

``` JavaScript
const order = { type = 'coffee' }

// const 允许更改 order 的 type
order.type = 'tea' // 这样没关系

// const 不允许给 `order` 重新分配引用
order = { type: 'tea' } // 这样会出错
```

我喜欢用 `const` 来提醒我自己一个对象或者数组不应该被更改（大多数情况下都是这样的）。如果写代码的时候，我明确知道我会更改某个对象或者数组，我会用 `let` 声明它。这只是一个习惯性的规则。

## 在 Redux 中要如何更新 state 呢？

Redux 要求 reducers 都是纯函数。也就是说，你不能直接更改 state，你必须基于原本的 state 创建一个新的 state，就像我们上面对 Bob 做的一样。（如果你不相信，可以看看 [reducer 是什么](https://daveceddia.com/what-is-a-reducer/)以及这个名字的来源是什么。）

写不可变的状态更新可能很复杂。下面，你可以找到一些常见的模式。

可以在浏览器控制台或者实际项目里自己动手试试。我觉得嵌套对象的更新是最难得，要特别注意并多加练习。

这些实际上也全部适用于 React 的 state，所以你在这份教程里学到的不仅仅适用于 Redux。

在最后，我们会看看怎么使用 Immer 库让这些更方便——但是不要直接跳到结尾！如果你打算使用现有的代码库，那么理解原理非常有用处。

## ... 扩展操作符

很多例子经常对对象或者数组使用扩展操作符，以下是它的执行过程。

当 `...` 符号被写在对象或者数组的前面，它会展开对象或者数组的子属性，然后就把他们插入到原地。

``` JavaScript
// 数组
let nums = [1, 2, 3]
let newNums = [...nums] // [1, 2, 3]
nums === newNums // false。newNums 是一个新的数组。

// 对象
let person = { name: 'Liz', age: 32 }
let newPerson = { ...person }
person === newPerson // false。newPerson 是一个新的对象。

// 内部属性保持不变
let company = {
  name: 'Foo Corp',
  people: [{ name: 'Joe' }, { name: 'Alice' }],
}
let newCompany = { ...company }
company === newCompany // false。newCompany 是一个新的对象。
company.people === newCompany.people // true
```

像上面这样，扩展操作符很方便就能创建一个和另一个对象/数组属性相同的新对象/数组。你可以轻松创建一个对象/数组的副本，然后重写你需要改变的特定属性：

``` JavaScript
let liz = {
  name: 'Liz',
  age: 32,
  location: {
    city: 'Portland',
    state: 'Oregon',
  },
  pets: [{ type: 'cat', name: 'Redux' }],
}

// liz 长大了一岁，其他的信息都没有改变
let olderLiz = {
  ...liz,
  age: 33,
}
```

从 ES2018 开始，扩展操作符是标准 JavaScript 的一部分。

## 更新 state 的诀窍

这些例子写的都是 Redux reducer，我会展示传入的 state 的样子，然后展示如何返回一个更新过的 state。

为了让例子更简洁，我会完全省略掉”action“参数。我们假装 state 更新会在所有 action 到来时触发。当然，在你自己的 reducers 中，可能会有 `switch` 声明以及针对每个 action 的 `case`，在这里我也都省略掉了。

### 在 React 中更新组件中的 state

如果要把这些例子用在更新 React 组件中的 state 上，只需要在例子上简单调整一下。

因为 React 会浅合并你传入 `this.setState()` 中的对象，你没必要像在 Redux 中那样用扩展操作符操作原本的 state。

在 Redux reducer 中，你可能这样写：

``` JavaScript
return {
  ...state,
  (updates here),
}
```

如果要更新组件中的 state，只需要这样写：

``` JavaScript
this.setState({
  updates here,
})
```

记住，尽管 `setState` 会做一个浅合并，但如果你要更新 state 中深层嵌套的属性（只要比第一层深），你还是需要使用扩展操作符。

## Redux：更新对象

如果你想要更新 Redux state 的第一层属性，用 `...state` 复制原有的 state，然后列出你想要改变的属性和它们新的值就可以了。

``` JavaScript
function reducer(state, action) {
  /*
    State 的样子：

    state = {
      clicks: 0,
      count: 0
    }
  */

  return {
    ...state,
    clicks: state.clicks + 1,
    count: state.count - 1,
  }
}
```

## Redux: 更新对象中的对象

（这不是 Redux 特有的，同样的方法也适用于组件中的 state。）

如果你想要更新的对象没有在 Redux state 的第一层，你需要制作每一层的副本，包括你想要更新的那个对象。下面是例子：

``` JavaScript
function reducer(state, action) {
  /*
    State 的样子：

    state = {
      house: {
        name: 'Ravenclaw',
        points: 17,
      }
    }
  */

  // 给拉文克劳加两分
  return {
    ...state,
    house: {
      ...state.house,
      points: state.house.points + 2,
    }
  }
}
```

另一个例子，更新更深一层：

``` JavaScript
function reducer(state, action) {
  /*
    State 的样子：

    state = {
      school: {
        name: 'Hogwarts', // 复制 state
        house: {
          name: 'Ravenclaw', // 复制嵌套的 object
          points: 17
        }
      }
    }
  */

  // 给拉文克劳加两分
  return {
    ...state, // 复制 state
    school: {
      ...state.school, // 复制第一层
      house: {         // 替换 state.school.house
        ...state.school.house, // 复制原本 house 的属性
        points: state.school.house.points + 2  // 更改属性
      }
    }
  }
```

在更新深层嵌套的属性时，代码会变得很难阅读。

## Redux：按照属性名更新对象

（这不是 Redux 特有的，同样的方法也适用于组件中的 state。）

``` JavaScript
function reducer(state, action) {
  /*
    State 的样子：

    const state = {
      houses: {
        gryffindor: {
          points: 15
        },
        ravenclaw: {
          points: 18
        },
        hufflepuff: {
          points: 7
        },
        slytherin: {
          points: 5,
        }
      }
    }
  */

  // 给拉文克劳加三分，学院名已经被存入了一个变量中
  const key = 'ravenclaw'
  return {
    ...state, // 复制 state
    houses: {
      ...state.houses, // 复制 houses
      [key]: { // 更新具体的 house
        ...state.houses[key], // 复制 house 的属性
        points: state.houses[key].points + 3, // 更新 point 属性
      }
    }
  }
}
```

## Redux：在数组头部插入一项

（这不是 Redux 特有的，同样的方法也适用于组件中的 state。）

如果要用可变的方法做这件事，就是使用数组的 `.unshift()` 方法在前面增加一项。`Array.prototype.unshift` 会改变数组，这不是我们想看到的。

下面是在数组头部插入一项的不可变的做法，适用于 Redux：

``` JavaScript
function reducer(state, action) {
  /*
    State 的样子：

    state = [1, 2, 3];
  */

  const newItem = 0
  return [ // 一个新数组
    newItem, // 在头部增加一项
    ...state, // 把原本的 state 在尾部展开
  ]
}
```

## Redux：在数组尾部插入一项

（这不是 Redux 特有的，同样的方法也适用于组件中的 state。）

做这件事的可变方法是使用数组的 `.push` 方法在尾部增加一项。但是会改变数组。

下面是在数组尾部插入一项的不可变的做法：

``` JavaScript
function reducer(state, action) {
  /*
    State 的样子：

    state = [1, 2, 3];
  */

  const newItem = 0
  return [ // 新数组
    ...state, // 先把原本的 state 展开
    newItem, // 把新的一项插入到最后
  ]
}
```

你也可以先用 `.slice` 创建一个数组的副本，然后改变这个副本：

``` JavaScript
function reducer(state, action) {
  const newItem = 0
  const newState = state.slice()

  newState.push(newItem)
  return newState
}
```

## Redux：使用 `map` 更新数组中的一项

（这不是 Redux 特有的，同样的方法也适用于组件中的 state。）

数组的 `.map` 方法会返回一个新的数组。其规则是，调用你提供的函数，把数组中的每一项传入函数，然后把返回值作为这一项新的值。

也就是说，如果你有一个 N 项的数组，然后想要一个仍旧有 N 项的新数组，使用 `.map`。你可以利用传入的函数更新/替换一项或是多项。

``` JavaScript
function reducer(state, action) {
  /*
    State 的样子：

    state = [1, 2, 'X', 4]
  */

  return state.map((item, index) => {
    // 把 X 替换为 3
    // alternatively: you could look for a specific index
    if(item === 'X') {
      return 3
    }

    // 不改变其他项
    return item
  })
}
```

## Redux：更新数组中的一个对象

（这不是 Redux 特有的，同样的方法也适用于组件中的 state。）

这和上面的做法是一样的。唯一的不同是，我们需要构建一个新的对象，然后返回我们想要更新的那个对象的副本。

在这个例子中，我们有一个用户邮箱信息的数组。有一些用户的邮箱已经更换了，我们需要更新信息。我会展示用户的 id 和邮箱是如何作为 `action` 的一部分传入的，你当然也可以调整一下，接受来自其他地方的值（如果你没有在用 Redux）。

``` JavaScript
function reducer(state, action) {
  /*
    State 的样子：

    state = [
      {
        id: 1,
        email: 'jen@reynholmindustries.com',
      },
      {
        id: 2,
        email: 'peter@initech.com',
      }
    ]

    action 中包含了新的信息：

    action = {
      type: 'UPDATE_EMAIL'
      payload: {
        userId: 2,  // Peter's ID
        newEmail: 'peter@construction.co'
      }
    }
  */

  return state.map((item, index) => {
    // 找到 id 对应的那一项
    if(item.id === action.payload.userId) {
      // 返回一个新的对象
      return {
        ...item,  // 复制原本的 item
        email: action.payload.newEmail  // 替换邮箱地址
      }
    }

    // 不改变其他项
    return item;
  });
}
```

## Redux：在数组中间插入一项

（这不是 Redux 特有的，同样的方法也适用于组件中的 state。）

数组的 `.splice` 方法可以在数组中间插入一项，但是也会改变数组本身。

既然我们不想改变原始数组，我们可以先创建一个副本（用 `.slice`），然后用 `.splice` 在副本的中间插入一项。

另一种方法是，先复制新元素前面的所有元素，然后插入新的一项，然后复制新元素后面的所有元素。但是这样很容易把序号搞错。

专业提示：这里是很容易犯错的，需要编写单元测试。

``` JavaScript
function reducer(state, action) {
  /*
    State 的样子：

    state = [1, 2, 3, 5, 6]
  */

  const newItem = 4

  // 创建一个副本
  const newState = state.slice()

  // 在数组中间插入新的一项
  newState.splice(3, 0, newItem)

  return newState

  /*
  // 你也可以这样做：

  return [                // 创建一个新的数组
    ...state.slice(0, 3), // 复制没有改变的前三项
    newItem,              // 插入新的一项
    ...state.slice(3)     // 复制剩下的，从 3 开始
  ]
  */
}
```

## Redux：按照 index 更新数组中的一项

（这不是 Redux 特有的，同样的方法也适用于组件中的 state。）

我们可以使用数组的 `.map` 方法为该 index 返回一个新的值，并保持其他项不变。

``` JavaScript
function reducer(state, action) {
  /*
    State 的样子：

    state = [1, 2, 'X', 4]
  */

  return state.map((item, index) => {
    // 替换 index 为 2 的值
    if (index === 2) {
      return 3
    }

    // 不改变其他项
    return item
  })
}
```

## Redux：使用 `filter` 从数组中移除元素

数组的 `.filter` 方法会调用你提供的函数，传入数组中的每一项。它会返回一个新的数组，只包括你的函数返回 true 的项目。

如果有一个 N 项的数组，你只想保留一部分，使用 `.filter`。

``` JavaScript
function reducer(state, action) {
  /*
    State 的样子：

    state = [1, 2, 'X', 4]
  */

  return state.filter((item, index) => {
    if(item === 'X') {
      return false
    }

    return true
  })
}
```

在 Redux 文档[不可变的更新模式](https://redux.js.org/recipes/structuring-reducers/immutable-update-patterns)的部分中还有其他好的技巧。

## 使用 Immer 轻松更新 state

如果你因为上面不可变更新状态的代码想要尖叫着逃跑，我不会怪你的。

深层嵌套的对象更新难阅读，难书写，还难以写对。单元测试在此时十分有必要，但就算这样也不能让这种代码变得好写好读。

幸好有一个库可以帮助我们。使用 Michael Weststrate 写的 [Immer](https://github.com/mweststrate/immer)，你可以写你熟悉又喜欢的可变代码，`[].push`、`[].pop` 还有 `=` 之类你都可以用——Immer 会魔术般接管代码，生产出完美的不可变更新。

这个真的很厉害。让我们看看它是怎么回事：

首先，你需要安装 Immer。（只有 2K，小但是很厉害。）

``` JavaScript
yarn add immer
```

然后，你需要从 Immer 引入 `produce` 函数。它只有这一个出口。这个函数就是它做的所有事。多么棒、美好而专注。

``` JavaScript
import produce from 'immer'
```

顺便一提，它的名字叫“produce”是因为它创造出了新的值，这个名字与“reduce”正相反。这里有一个他们最初讨论名字的 [issue](https://github.com/mweststrate/immer/issues/24)。

这样，你就可以使用 `produce` 函数为自己创造出一个小小的可变游乐场，你做出的所有改变都会被 JS 中转的魔法所处理。以下是对比，先是直接用 JS 更新一个嵌套在对象内部的值，然后是用 Immer：

``` JavaScript
/*
  State 的样子：

  state = {
    houses: {
      gryffindor: {
        points: 15
      },
      ravenclaw: {
        points: 18
      },
      hufflepuff: {
        points: 7
      },
      slytherin: {
        points: 5
      }
    }
  }
*/

function plainJsReducer(state, action) {
  // 给拉文克劳加三分
  // 学院名存在变量中
  const key = 'ravenclaw'
  return {
    ...state, // 复制 state
    houses: {
      ...state.houses, // 复制学院
      [key]: {
        ...state.houses[key],  // 复制该学院的全部属性
        points: state.houses[key].points + 3   // 更新 points 属性
      }
    }
  }
}

function immerifiedReducer(state, action) {
  const key = 'ravenclaw'

  // produce 接受原本的 state，以及一个 function
  // 它会调用这个 function，并向其中传入一个草稿版本的 state
  return produce(state, draft => {
    // 随便改这个草稿
    draft.houses[key].points += 3

    // 更改过的草稿会被自动 return，不需要手动 return 任何东西
  })
}
```

## 对 React 组件中的 state 使用 Immer

Immer 对于组件中的 state 也很有用——通过函数形式的 setState。

你可能已经知道 React 的 `setState` 有一个函数形式，可以接受一个函数，并向这个函数传入当前的 state。这个函数会返回新的 state：

``` JavaScript
onIncrementClick = () => {
  // 普通形式
  this.setState({
    count: this.state.count + 1,
  })

  // 函数形式
  this.setState(state => {
    return {
      count: state.count + 1,
    }
  })
}
```

Immer 的 `produce` 方法可以被作为 state 的更新函数。你可能会注意到，在这种情况下调用 `produce` 只传入一个参数——更新函数——而不像 reducer 的例子里一样传入两个参数 `(state, draft => {})`。

``` JavaScript
onIncrementClick = () => {
  this.setState(produce(draft => {
    draft.count += 1
  })
}
```

## 逐渐采用 Immer

Immer 的优点是，它非常小而专注（只暴露出一个函数，用于创建新的 state），很容易将其加入到现有的代码库中进行尝试。

Immer 也向下兼容已有的 Redux reducers。如果你把已有的 `switch/case` 包在 Immer 的 `produce` 函数中，你对 reducers 的测试仍都可以通过。

之前我展示过，传递给 `produce` 的更新函数可以只返回 `undefined`，`produce` 会自动收集你对于 `draft` 的更新。我没有提到的是，其实更新函数也可以返回一个全新的 state，只要你并没有对 `draft` 进行过更改。

也就是说，已有的会返回全新 state 的 Redux reducers，也可以被包在 Immer 的 `produce` 方法中，效果是一样的。因此，你可以在你有空的时候慢慢替换难以阅读的不可变代码。查看官方例子：[从 producers 返回数据的不同方法](https://github.com/mweststrate/immer#returning-data-from-producers)。

## 原文

[Immutability in React and Redux: The Complete Guide](https://daveceddia.com/react-redux-immutability-guide/)
