---
title: Underscore.js实现bind方法
date: 2018-07-24 21:02:28
categories: [框架/库/工具, Underscore.js]
tags: [JavaScript, Underscore.js]
---

## ES5中的bind

### 描述

bind()函数会创建一个新函数（称为绑定函数），新函数与被调用函数（绑定函数的目标函数）具有相同的函数体（。当新函数被调用时，this值绑定到bind()的第一个参数，该参数不能被重写。绑定函数被调用时，bind()也接受预设的参数提供给原函数。
一个绑定函数也能够使用new操作符创建对象，这种行为就像是把原函数当做构造器。提供的this值被忽略，同时调用时的参数被提供给模拟函数。

### 语法

> fun.bind(thisArg[, arg1, [, arg2, [, …]]])
> 
> - thisArg：当调用绑定函数时，该参数会作为目标函数运行时的this指向。当使用new操作符调用绑定函数时，该参数无效。
> - arg1, arg2, …：当调用目标函数时，前置于提供给绑定函数的参数的参数
> - 返回值：返回由指定的this值和初始化参数改造的原函数拷贝

### 示例

#### 创建绑定函数

bind()最简单的用法是创建一个函数，使这个函数无论怎么调用都有同样的this值。

```
this.x = 9

var module = {
  x: 81,
  getX: function () {
    return this.x
  }
}

module.getX() // 81

var retrieveX = module.getX()
retrieveX() // 9

var boundGetX = retrieveX.bind(module)
boundGetX() // 81
```

#### 偏函数（partial functions）

bind()的另一个简单用法是使一个函数具有预设的初始参数。这些参数会被插入到目标函数的参数列表的开始位置，传递给绑定函数的参数会跟在它们的后面。

```
function list () {
  return Array.prototype.slice.call(arguments)
}

var list1 = list(1, 2, 3) // [1, 2, 3]
var leadingThirtysevenList = list.bind(null, 37)
var list2 = leadingThirtysevenList() // [37]
var list3 = leadingThirtysevenList(1, 2, 3) // [37, 1, 2, 3]
```

#### 绑定函数作为构造函数使用

绑定函数当然也可以使用new操作符去构造目标函数创建的新实例。当绑定函数被用于构造一个值时，提供的this值被忽略。然而，提供的参数预先提供给构造器。

```
function Point (x, y) {
  this.x = x
  this.y = y
}

Point.prototype.toString = function () {
  return this.x + ', ' + this.y
}

var p = new Point(1, 2)
p.toString() // 1, 2

var emptyObj = {}
var YAxisPoint = Point.bind(emptyObj, 0)
var axisPoint = new YAxisPoint(5)

axisPoint.toString() // 0, 5
axisPoint instanceof Point // true
axisPoint instanceof YAxisPoint // true
new Point(17, 42) instanceof YAxisPoint; // true

// 仍可以作为普通函数被调用
YAxisPoint(13)
empty.x + ', ' + empty.y // 0, 13
```

## underscore实现bind

### 最简单的方法

也是一开始我自己写出的方法：

```
_.bind = restArgs(function (func, context, args) {
  return restArgs(function (callArgs) {
    return func.apply(context, args.concat(callArgs))
  })
})
```

本质上，实际上返回一个将目标函数包裹起来的绑定函数，利用闭包储存了context，在执行绑定函数时，就执行目标函数，同时将利用apply将this置为context。
但这个方法不能够处理用new调用的情况。根据ES5标准，使用new调用bind函数，创建的应该是目标函数的实例。但是此时，使用new创建的显然是绑定函数的实例。

### 解决使用new创建实例的问题

underscore是这样解决这个问题的：

```
var baseCreate = function (obj) {
  var Ctor = function () {}
  Ctor.prototype = obj
  return new Ctor()
}

var executeBound = function (sourceFn, boundFn, context, callingContext, args) {
  // 如果不是使用new调用函数，则按照原操作处理
  if (!(this instanceof callingContext)) {
    return func.apply(context, args)
  }

  var self = baseCreate(sourceFn.prototype)
  var result = sourceFn.apply(self, args)
  if (_.isObject(result)) return result
  return self
}

_.bind = restArgs(function (func, context, args) {
  var bound = restArgs(function (callArgs) {
    return executeBound(func, bound, context, this, args.concat(callArgs))
  })
  return bound
})
```

同样，是执行绑定函数，返回目标函数的执行结果。不同的是，要先判断是否是new调用。
如果不是new调用，则正常返回目标函数的执行结果。
如果是new调用，则创建一个[[prototype]]指向目标函数原型对象的实例，相当于模拟new执行函数的第一步，然后将这个实例放入目标函数中执行，this指向该实例，模拟第第二步，最后返回该实例，模拟第三步。
不同的是，underscore做的另一个处理是，如果目标函数本来就返回一个对象，那么执行结果就不返回该实例，而是返回目标函数本身返回的对象。
