---
title: Underscore.js源码笔记
date: 2017-11-23 15:29:56
categories: [框架/库/工具, Underscore.js]
tags: [JavaScript, Underscore.js, 源码]
---

## 用void 0代替undefined

### undefined可以被重写

1. undefined并不是保留词（reserved word），只是全局对象的一个属性（`window.undefined === undefined`），因此在低版本的IE中可以被重写。
    
``` JavaScript
var undefined = 10
console.log(undefined) // 10 -- IE8
```

1. undefined在ES5中已经是一个只读（read-only）属性了，不能被重写。但是在局部作用域中，还是可以被重写的。与浏览器版本无关。

``` JavaScript
(function () {
  var undefined = 10
  console.log(undefined) // 10
})()
```

``` JavaScript
（function () {
  undefined = 10
  console.log(undefined) // undefined
}）()
```

<!-- more -->

### 用void 0代替undefined的原因

1. MDN解释：
> The void operator evaluates the given expression and then returns undefined.
> 即void运算符对给定的表达式进行求值，返回undefined。也就是说，void后跟随任何表达式，返回值都是undefined。而void是不能被重写的（cannot be overidden）。

2. void 0代替undefined能够节省字节大小，很多JavaScript压缩工具都是将undefined用void 0替换掉的。

### 原文与参考

- [为什么用“void 0”代替“undefined” - hanzichi](https://github.com/hanzichi/underscore-analysis/issues/1)
- [void - MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/void)
- [What does ‘void 0’ mean? - stackoverflow](https://stackoverflow.com/questions/7452341/what-does-void-0-mean)

## 字符串强制转换为数字

原Underscore.js代码不对字符串格式的fromIndex做处理，但考虑到原生方法会做，就自己加了。

开始用的是Number()，但该方法会将null、’’、true、false等都转换为数字，与预期不符。

parseInt()和parseFloat()则不同，只有对String类型调用这两个方法才能正确运行，对其他类型返回的都是NaN。会从string开头识别整数（浮点数），非整数（浮点数）部分省略，也与预期不符。’123a’ -> ‘123’。

最终决定先判断是否是字符串，字符串是否为空，然后使用Number()类型转换。’123a’ -> ‘NaN’

## 关系操作符

在写Underscore.js时发现，如果数字与非数字的字符串比较大小，无论大于还是小于都是false。翻看比较操作符规则：

- 如果两个操作数都是数字，则执行数值比较；
- 如果两个操作数都是字符串，则比较两个字符串对应的字符编码值；
- 如果一个操作数是数值，则将另一个操作数转换为一个数值，然后执行数值比较；（1与’a’比较，’a’被转换为NaN，因此’a’不大于不小于不等于1）
- 如果一个操作数是对象，则调用这个对象的valueOf()方法，用得到的结果按照前面的规则执行比较。如果对象没有valueOf()方法，则调用toString()方法，并用得到的结果根据前面的规则执行比较；
- 如果一个操作数是布尔值，则先将其转换为数值，再进行比较。（false < true）

### 原文与参考

- 《JavaScript高级程序设计》，page 50，3.5.6 关系操作符

## for…in存在的浏览器兼容性问题

### for…in

循环遍历可枚举属性，包括Array/Object对象本身的可枚举属性和prototype中的可枚举属性。prototype中的可枚举属性即自定义属性。

### 在IE<9中的bug

在IE8中，将[‘valueOf’, ‘isPrototypeOf’, ‘toString’, ‘propertyIsEnumerable’, ‘hasOwnProperty’, ‘toLocaleString’]等内定为了“不可枚举属性”，即使该属性已经被重写。

``` JavaScript
var obj = { toString: 'wangyi' }
for (var key in obj) {
  console.log(key) // 不打印出任何东西
}
```

Underscore.js解决这个问题的处理是：

``` JavaScript
var hasEnumBug = !{toString: null}.propertyIsEnumerable('toString');
var nonEnumerableProps = ['valueOf', 'isPrototypeOf', 'toString', 'propertyIsEnumerable', 'hasOwnProperty', 'toLocaleString'];

var collectNonEnumProps = function(obj, keys) {
  var nonEnumIdx = nonEnumerableProps.length;
  var constructor = obj.constructor;
  var proto = _.isFunction(constructor) && constructor.prototype || ObjProto;

  // Constructor is a special case.
  var prop = 'constructor';
  if (_.has(obj, prop) && !_.contains(keys, prop)) keys.push(prop);

  while (nonEnumIdx--) {
    prop = nonEnumerableProps[nonEnumIdx];
    if (prop in obj && obj[prop] !== proto[prop] && !_.contains(keys, prop)) {
      keys.push(prop);
    }
  }
};
```

其中出现了两种判断方法。一种是prop in obj && obj[prop] !== proto[prop]，即判断该属性存在于obj中（本身或原型链），且本身与原型链中的属性值不同。

第二种判断方法被针对于constructor，即用hasOwnProperty判断obj本身是否有constructor属性，有则推入。

个人认为第一种方法有漏洞，如果重写了某属性且属性值本来就与原型链中的属性值相同，则无法判断该属性已经被重写。第二种方法才是根源上收集了obj中所有的属性。

关于为什么第二种方法只用于处理了constructor，考虑到两段代码并不是同一个人所写，怀疑只是第二个人觉得能修补就不需要修改了。

## sample（取随机数）

关于取随机数，且不重复取，写法很聪明了，将取过的放在开头，然后每次越过开头取随机index。最终结果直接从开头取既定位数就可以

## sort

如果省略sort的compareFunction，那么元素会按照转换为的字符串的诸个字符的Unicode位点进行排序，即“80”比“9”靠前。
