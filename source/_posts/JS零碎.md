---
title: JS零碎
date: 2017-03-08 15:35:18
categories: [JavaScript, 基础]
tags: [JavaScript]
---

## 浅复制和深复制

针对JS引用类型值（Object、Array），浅复制只复制原对象的一层属性，如果引用类型值中的一个值仍旧是引用类型值，会导致复制出的对象与原对象的这个值引用同一个地址。

深复制不仅将原对象的各个属性逐个复制，而且将原对象各个属性所包含的对象也采用深复制的方法递归复制到新对象上。

深复制递归实现：

```
function copyObj(obj) {
  if (obj.constructor == Array) {
    return obj.slice()
  }

  var newObj = {}
  for (var key in obj) {
    if (obj[key].constructor == Array || obj[key].constructor == Object) {
      newObj[key] = copyObj(obj[key])
    } else {
      newObj[key] = obj[key]
    }
  }
  return newObj
}
```

<!-- more -->

## call与apply的区别

功能上，都是调用一个对象的方法，以另一个对象替换当前对象。仅传递的参数不同，call从第二个参数开始作为方法的参数传入，而apply传入的是一个参数数组。使用apply的好处是可以直接将当前函数的arguments对象作为apply的第二个参数传入。

## 函数防抖动

滚动、窗口放大缩小、键盘按下事件时，可能在短时间内触发多次，必须限制一段时间后才能再次触发函数。

【scroll事件是在window对象上发生的，也可以用在有滚动条的元素上。】

```
function debounce(fn, delay) {
  var timer = null
  return function () {
    clearTimeout(timer)
    var context = this,
      args = arguments
    timer = setTimeout(function () {
      fn.apply(context, args)
    }, delay)
  }
}

document.getElementById('container').addEventListener('scroll', debounce(function () {
  console.log('scroll')
}, 200))
```

## 事件委托

利用了事件冒泡/捕获的原理，把事件监听绑定在父元素或祖元素上，避免重复查找元素，避免绑定多个事件处理函数占用内存，优化性能。

```
<ul>
  <li></li>
  <li></li>
  <li></li>
</ul>

<script>
  document.querySelector('ul').addEventListener('click', function (event) {
    if (event.target.nodeName == 'LI') {
      event.target.innerHTML = 'clicked'
    }
  })
</script>
```

虽然可以将所有事件都绑定在document上，但是有可能被其他元素干扰，如需要事件监听的元素的某祖先元素需要e.stopPropagation()，那么事件达不到document就会被阻止掉。除此之外，时间冒泡也需要耗时，越靠近顶层耗时越长，影响反应时间。

## 函数声明和函数表达式的区别

在进行变量提升时，函数声明的提升包括函数体，且在预执行期就已经执行了函数声明，在脚本中实际声明的位置不会重复执行。而函数表达式和其他类型的变量提升一样，只提升变量的声明。因此，将会出现如下两种情况：

```
fn()
function fn () {
  console.log('fn')
}
// 输出fn

fn()
var fn = function () {
  console.log('fn')
}
// fn is not a function
// 函数声明的提升包括函数体
```

```
fn ()
if (true) {
  function fn () {
    console.log('true')
  }
} else {
  function fn () {
    console.log('false')
  }
}
// 输出false
// 函数声明提升，且在预编译阶段已经执行，因此后一条声明覆盖了前一条，无法辨别if语句。也是规范不允许函数声明出现在循环、条件判断、try/finally以及with语句中
// 但其实目前浏览器已经不支持这种语法，提示fn is not a function
```

## JS赋值顺序

```
var a = {n: 1}
var b = a
a.x = a = {n: 2}
// a -> {n:2}
// b -> {n:1, x: {n:2}}
```

先从左到右将每个引用替换为真实的对象的属性，再从右到左计算。因此最左边的foo.x一开始就被替换为{n:1}的x属性的引用，赋值也是赋给了这个对象的引用。

## DOM事件流

“DOM2级事件”规定的事件流包括三个阶段：事件捕获阶段、处于目标阶段和事件冒泡阶段。首先发生的是事件捕获，为截获事件提供了机会。然后是实际的目标接收到事件。最后一个阶段是冒泡阶段，可以在这个阶段对事件作出响应。
![DOM2级事件](http://wx3.sinaimg.cn/mw690/7b1152ffly1fg5qr5p2bej20lk0eognj.jpg)
在DOM事件流中，实际的目标（div元素）在捕获阶段不会接收到事件。这意味着在捕获阶段，事件从document到html再到body就停止了。下一个阶段是“处于目标阶段”，于是事件在div上发生，并在事件处理中被看成冒泡阶段的一部分。然后，冒泡阶段发生，事件又传播回文档。
**IE8及更早版本不支持DOM事件流，而是事件冒泡。**

### DOM0级事件处理程序

每个元素（包括document和window）都有自己的事件处理程序属性，这些属性通常全部小写，例如onclick。将这种属性的值设置为一个函数，就可以指定事件处理程序。

```
// 使用HTML指定事件处理程序
<button id="myBtn" onclick="alert('Clicked!')">Button</button>

// 或者
<button id="myBtn">Button</button>
var btn = document.getElementById('myBtn')
btn.onclick = function () {
  alert('Clicked!')
}
```

使用DOM0级方法指定的事件处理程序被认为是元素的方法。因此，事件处理程序中的this引用当前元素。

（如果浏览器支持DOM事件流，）以这种方式被添加的事件处理程序会在事件流的冒泡阶段被处理。

```
btn.onclick = null // 删除通过DOM0级方法指定的事件处理程序
```

### DOM2级事件处理程序

“DOM2级事件”定义了两个方法，用于处理指定和删除事件处理程序的操作：addEventListener()和removeEventListener()。所有DOM节点都包含这两个方法，并且它们都接受三个参数：要处理的事件名、作为事件处理程序的函数和一个布尔值。最后的布尔值如果是true，表示在捕获阶段调用事件处理程序；如果是false，表示在冒泡阶段调用事件处理程序。

与DOM0级方法一样，这里添加的事件处理程序的this引用其依附的元素。

使用DOM2级方法添加事件处理程序的主要好处是可以添加多个事件处理程序。事件处理程序会按照添加它们的顺序触发。

通过addEventListener()添加的事件处理程序只能使用removeEventListener()移除，移除时传入参数与添加处理程序时使用的参数相同。这意味着通过addEventListener()添加的匿名函数将无法移除。

```
var btn = document.getElementById('myBtn')

var handler = function () {
  alert('Clicked!')
}

btn.addEventListener('click', handler, false)
btn.removeEventListener('click', handler, false)
```    

**IE8不支持DOM2级事件处理程序。**

### IE事件处理程序

IE实现了与DOM中类似的两个方法：attachEvent()和detachEvent()。这两个方法接收同样的参数：事件处理程序名称和事件处理程序函数。由于IE8及更早版本只支持事件冒泡，所以通过attachEvent()添加的事件处理程序都会被添加到冒泡阶段。

attachEvent()的第一个参数是“onclick”，而不是DOM的addEventListener()方法的“click”。

在IE中使用attachEvent()与DOM0级方法的主要区别在于事件处理程序的作用域。在使用DOM0级方法的情况下，事件处理程序会在其所属元素的作用域内运行；在使用attachEvent()方法的情况下，事件处理程序的this等于window。

```
var btn = document.getElementById('myBtn')
btn.attachEvent('onclick', function () {
  alert(window === this) // true
})
```

与addEventListener()方法类似，attachEvent()也可以用来为一个元素添加多个事件处理程序。不过，与DOM方法不同的是，这些事件处理程序不是以添加它们的顺序执行，而是以相反的顺序被触发。

使用detachEvent()方法可以移除attachEvent()添加的事件，条件是提供相同的参数。与DOM方法一样，这也意味着添加的匿名函数不能被移除。

## 使用DocumentFragment将DOM分批插入页面

DocumentFragment节点不属于文档树，继承的parentNode节点总是null。不过它有一个特殊行为，使它十分有用，即：当请求将一个DocumentFragment节点插入文档树时，插入的不是DocumentFragment节点自身，而是它的所有子孙节点。这使得DocumentFragment成了有用的占位符，暂时存放那些一次插入文档的节点，还有利于实现文档的剪切、复制和粘贴操作。

```
(() => {
  var ndContainer = document.getElementById('js-list')
  if (!ndContainer) {
    return
  }

  const total = 30000
  const batchSize = 4 // 每次插入的节点个数
  const batchCount = total / batchSize // 需要插入的次数
  var batchDone = 0

  function appendItems() {
    const documentFragment = new DocumentFragment()
    for (var i = 0; i < batchSize; i++) {
      const ndItem = document.createElement('li')
      ndItem.innerText = batchDone * batchSize + i
      documentFragment.appendChild(ndItem)
    }

    ndContainer.appendChild(documentFragment)
    batchDone++
  }

  while (batchDone < batchCount) {
    // 以页面重绘的频率插入就可以了
    window.requestAnimationFrame(appendItems)
  }

})()
```

## requestAnimationFrame

相当一部分浏览器的显示频率是一帧16.7ms（16.7ms == 1000 / 60），如果我们setTimeout为10ms更改画面，就会出现丢帧的情况。这也是setTimeout定时器推荐最小使用16.7ms的原因。

requestAnimationFrame就是为此而出现的，如果浏览器设备绘制的间隔为16.7ms，就按照这个间隔绘制。如果浏览器设备绘制间隔为10ms，就按照10ms绘制。

内部操作原理：浏览器（如页面）每次要重绘，就会通知requestAnimationFrame。

资源利用高效：

1. 即使是多个requestAnimationFrame并发，浏览器只需要通知一次，而setTimeout是多个独立绘制的；
2. 页面最小化或tab切换关闭，页面不再重绘，不会通知，则requestAnimationFrame也不再进行。

## encodeURI和encodeURIComponent

[https://www.cnblogs.com/season-huang/p/3439277.html](https://www.cnblogs.com/season-huang/p/3439277.html)

## String特点

字符串一旦创建，值就不能改变。要改变某个变量保存的字符串，首先要销毁原来的字符串，然后再用另一个包含新值的字符串填充变量。

```
var lang = 'Java'
lang = lang + 'Script'
```

首先创建一个能够容纳10个字符的字符串，填充’Java’和’Script’，销毁原本的字符串’Java’和新的字符串’Script’。

## 基本包装类型

为了便于操作基本类型值，ECMAScript还提供了三个特殊的引用类型值：Boolean、Number、String。每当读取一个基本类型值的时候，后台会自动创建一个基本包装类型的对象，从而让我们能够调用一些方法来操作这些数据。

```
var str1 = 'Some String'
var str2 = str1.substring(2)
```

变量str1包含一个字符串，字符串是基本类型值。而下一行调用了str1的substring方法，并将返回的结果保存在str2中。基本类型值不是对象，从逻辑上将没有方法。为了让我们实现这种直观的操作，要在内存中读取字符串时，后台进行了一系列操作：

1. 创建String类型的一个实例；
2. 在实例上调用指定的方法；
3. 销毁创建的String实例。

也可以显示创建基本包装类型的实例，对实例调用typeof会返回object。（相当于创建的是一个对象实例，而str = ‘Some String’只是单单创建了一个基本类型值。）

Object构造函数也会根据传入值的类型返回相应的基本包装类型实例，如：

```
var obj = new Object('some text')
console.log(obj instanceof String) // true
```
