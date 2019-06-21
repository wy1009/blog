---
title: LazyMan
date: 2017-02-20 15:18:50
categories: [JavaScript, 算法]
tags: [JavaScript, 算法, ES6]
---

## 队列解法

简单说，就是建立一个队列，在调用eat、sleep等方法时将所有执行任务的函数推入队列，然后在下一个事件循环中才开始依次执行队列中的任务。

写的过程中注意到的点：

不太清楚原作者为什么不直接将函数推入队列，而是推入一个立即执行函数返回的函数。猜测他是想要利用闭包，外部函数执行完毕后，内部函数仍可访问到外部函数的变量，使得下一个事件循环执行队列中的函数时仍旧可以访问到传入的参数。但其实单单将函数推入队列也是将内部函数保留了下来，内部函数为该被推入的函数，外部函数为构造函数/原型中的sleep、eat等函数，也形成了闭包。

setTimeout(0)，主要看了以下链接的文章：http://www.cnblogs.com/silin6/p/4333999.html

JavaScript引擎是基于事件驱动单线程执行的，JS引擎一直等待着任务队列中任务的到来，然后加以处理，浏览器无论什么时候都只有一个JS线程在运行JS程序。

<!-- more -->

例：

``` JavaScript
var isEnd = true
window.setTimeout(function () {
  isEnd = false
}, 1000)
while(isEnd)
alert('end')
```

浏览器会假死，alert永远不会跳出。因为while(true)永远地占据了JS线程。

简单说，setTimeout(0)可以将任务留在所有没有设置延时的代码执行完毕之后再开始执行。

根据思想写出的代码：

``` JavaScript
function _LazyMan (name) { // lazyman构造函数
  this.tasks = []
  var self = this
  var fn = function () {
    console.log('Hi! This is ' + name + '!')
    self.next()
  }
  this.tasks.push(fn)
  setTimeout(function () {
    self.next()
  }, 0)
}
_LazyMan.prototype.next = function() {
  var fn = this.tasks.shift()
  fn && fn()
}
_LazyMan.prototype.sleep = function (second) {
  var self = this
  var fn = function () {
    setTimeout(function () {
      console.log('Wake up after ' + second + ' seconds')
      self.next()
    }, second * 1000)
  }
  this.tasks.push(fn)
  return this
}
_LazyMan.prototype.eat = function (name) {
  var self = this
  var fn = function () {
    console.log('Eat ' + name)
    self.next()
  }
  this.tasks.push(fn)
  return this
}
new _LazyMan('Siren').eat('dinner').sleep(10)
```

## Promise解法

简单说还是先建立一个队列，然后在最后一步，队列解法是将方法一个一个推出队列执行，而Promise解法则是将Promise的then串联起来。then方法返回的是一个新的Promise实例，因此可以采用链式调用。

then的链式调用有两种情况：

如果第一个then方法的回调函数返回的不是一个Promise对象，那么第一个then方法的回调函数执行完之后，会将返回结果作为参数，传入第二个then方法的回调函数。

如果第一个then方法的回调函数返回的还是一个Promise对象，这时后一个回调函数，就会等待该Promise对象的状态发生改变后，才会被调用。后一个回调函数的参数是前一个回调函数的Promise的resolve和reject传入的参数。

Promise解法代码：

``` JavaScript
function _LazyMan (name) {
  var makePromise = function () {
    var promise = new Promise(function (resolve, reject) {
      console.log('Hi! I am ' + name + '!')
      resolve()
    })
    return promise
  }
  this.promiseMakerList = []
  this.promiseMakerList.push(makePromise)
  var self = this
  var promiseTmp = new Promise(function (resolve, reject) {
    resolve()
  })
  setTimeout(function () {
    for (var i = 0; i < self.promiseMakerList.length; i ++) {
      var thenFn = (function (nowPromiseMaker) {
        return nowPromiseMaker
      })(self.promiseMakerList[i])
      promiseTmp = promiseTmp.then(thenFn)
    }
  }, 0)
}
_LazyMan.prototype.eat = function (name) {
  var makePromise = function () {
    var promise = new Promise(function (resolve, reject) {
      console.log('eat ' + name + '~')
      resolve()
    })
    return promise
  }
  this.promiseMakerList.push(makePromise)
  return this
}
_LazyMan.prototype.sleep = function (time) {
  var makePromise = function () {
    var promise = new Promise(function (resolve, reject) {
      setTimeout(function () {
        console.log('Wake up after ' + time + ' seconds!')
        resolve()
      }, time * 1000)
    })
    return promise
  }
  this.promiseMakerList.push(makePromise)
  return this
}
new _LazyMan('Wang Yi').sleep(1).eat('dinner')
```
