---
title: Underscore.js实现链式调用
date: 2017-12-14 20:11:13
categories: [框架/库/工具, Underscore.js]
tags: [JavaScript, Underscore.js]
---

## underscore基本结构

在全局变量上挂载一个对象“_”，将方法全部添加在这个对象上。

```
var root = typeof self == 'object' && self.self === self && self ||
  typeof global == 'object' && global.global === global && global ||
  this ||
  {}

var _ = {}
root._ = _
```

## 面向对象风格

调用underscore方法的两种方式：

```
_.each(obj, function () {})

// 面向对象风格
_(obj).each(function () {})
```

为实现面向对象风格，将`_`写为构造函数。

<!-- more -->

### 对`_`构造函数的处理

```
var _ = function (obj) {
  if (!(this instanceof _)) {
    return new _(obj)
  }

  this._wrapped = obj
}
```

意思很简单，在没有特殊设置的情况下，只有在使用`new`操作符调用函数时，函数内的`this`才会指向函数实例。因此根据`this`是否是`_`的实例可以确定是否是通过new调用`_`函数的。如果不是通过`new`调用，则通过`new`调用。
通过`new`调用，创建`_`的实例，其中实例的_wrapped属性为obj，即本应传入函数的值。

### 调用函数方法时的处理

在`_`构造函数中，生成了一个包含了*wrapped属性的实例。显然还并不能够实现直接通过该实例调用方法，因此，需要将方法加入到`*`所指向的原型对象中。

```
_.mixin = function (obj) {
  // _.functions为一个取出obj中所有方法名的函数
  _.each(_.functions(obj), function (name) {
    // 将obj的方法加入到`_`的属性中
    var func = _[name] = obj[name]
    _.prototype[name] = function () {
      // `_`的实例会调用的`_`的原型对象上的方法，而不是`_`的属性上的方法，因此可以直接通过this取到该实例
      var args = this._wrapped
      // apply可以将传入的array-like展开为一个个参数传入，使用apply调用push相当于将arguments一个个push入args中
      Array.prototype.push.apply(args, arguments)
      // 将func中的this置为`_`，使其表现与直接通过`_`调用的函数相同
      return func.apply(_, args)
    }
  })
}

_.mixin(_)
```

## 链式调用

### 表现

underscore的链式调用需要借助`_.chain`方法。

```
var lyrics = [
  'I\'m a lumberjack and I\'m okay',
  'I sleep all night and I work all day',
  'He\'s a lumberjack and he\'s okay',
  'He sleeps all night and he works all day',
]

var counts = _(lyrics).chain()
  .map(function (line) { return line.split('') })
  .flatten()
  .reduce(function (hash, l) {
    hash[l] = hash[l] || 0
    hash[l]++
    return hash
  }, {})
  .value()

// 也有这种写法
var counts = _.chain(lyrics).map()...
```

前一个方法的返回值是后一个方法的第一个参数。

### 实现

在underscore中，如果你选择了链式调用，那么你调用的所有方法都是`_`原型对象上的方法。这也方便了对返回结果的统一处理。
首先，将初始对象（即后续链式调用链上的方法的返回结果）通过`_.chain()`方法封装为一个带有_wrapped属性的实例，即前面所说的面向对象风格。同时通过`_chain`属性标记当前为链式调用模式。

```
_.chain = function (obj) {
  obj = _(obj)
  obj._chain = true
  return obj
}
```

然后通过该实例调用underscore的方法，这样，所有的方法都会经由`_`原型对象上的方法的“外包装”的统一处理，即以下部分：

```
_.prototype[name] = function () {
  // `_`的实例会调用的`_`的原型对象上的方法，而不是`_`的属性上的方法，因此可以直接通过this取到该实例
  var args = this._wrapped
  // apply可以将传入的array-like展开为一个个参数传入，使用apply调用push相当于将arguments一个个push入args中
  Array.prototype.push.apply(args, arguments)
  // 将func中的this置为`_`，使其表现与直接通过`_`调用的函数相同
  return func.apply(_, args)
}
```

再在这部分动些手脚：

```
_.prototype[name] = function () {
  // `_`的实例会调用的`_`的原型对象上的方法，而不是`_`的属性上的方法，因此可以直接通过this取到该实例
  var args = this._wrapped
  // apply可以将传入的array-like展开为一个个参数传入，使用apply调用push相当于将arguments一个个push入args中
  Array.prototype.push.apply(args, arguments)
  // 将func中的this置为`_`，使其表现与直接通过`_`调用的函数相同
  var obj = func.apply(_, args)
  return this._chain ? : _.chain(obj) : obj
}
```

这样，只要是通过`_`的实例所调用的方法，即`_`原型对象上的方法，其返回值都会被“外包装”处理。此时可以通过实例上的`_chain`标记，判断此时是否为链式调用。如果是链式调用，则再次调用`_.chain`方法，将返回值封装为实例的`_wrapped`属性，标记`_chain`属性，重复以上步骤。以此来将普通的返回值封入实例，进行下一步调用。

```
_.prototype.value = function () {
  return this._wrapper
}
```

最后，调用一个value()方法，结束链式调用。因为**链式调用实际上是把返回值封入实例，然后再通过实例调用方法**，因此，从实例中取出值，就是最终的返回值。

### 两种写法

第一种写法，`_(obj).chain()`，先创建实例，将`obj`存入`_wrapped`中。在调用`_.chain()`时，会先被“外包装”处理，将`_wrapped`中的`obj`取出，作为参数传入给`_.chain()`方法，相当于直接调用`_.chain(obj)`，然后由`_.chain(obj)`创建实例；
第二种写法，直接调用`_.chain(obj)`，只是跳过了之前的步骤。

### 总结

underscore链式调用使用的是其面向对象风格。根本思想是将`obj`封入实例，在借由实例调用方法前，先将`obj`从实例中“剥出来”，调用方法完成后，再将返回值“封回去”，再借由实例进行下一步调用。
可以分为两个部分，一是面向对象风格。首先将初始参数的obj进行面向对象处理，生成实例`{ _wrapped: obj }`，再借由实例调用方法。同时，实例调用的方法，也就是`_`原型对象上的方法，是经过了特殊处理的。为实现面向对象，将调用方法的实例中包裹的真正参数`_wrapped`整合其他传入方法的参数，传入方法。
二是链式调用。链式调用是在面向对象风格的基础上实现的。进行链式调用时，先调用`_.chain`，在实例上添加链式调用标记`_chain`。接下来，为实现链式调用，如果实例上有`_chain`，则将返回值也进行面向对象处理，同时标记`_chain`，使其能够继续以面向对象的风格调用，返回值也同样能调用方法，即实现了链式调用。一直到调用_.prototype.value，这个方法没有经过特殊处理，不会根据链式调用标记将返回值也进行面向对象处理，而是直接返回实例中的`_wrapped`，即最终返回值。
