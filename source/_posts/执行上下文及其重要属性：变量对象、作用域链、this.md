---
title: 执行上下文及其重要属性：变量对象、作用域链、this
date: 2017-04-13 11:10:18
categories: [JavaScript, 基础]
tags: [JavaScript]
---

## 执行上下文（executable contexts）与执行上下文栈

JavaScript的可执行代码（executable code）的类型有三种：全局代码、函数代码、eval代码。

当代码在执行过程中，遇到以上三种情况，都会生成一个执行上下文。执行上下文可以理解为当前代码的运行环境。

（每次某个函数被调用，就会有一个新的执行上下文为其创建，即使是调用自身的函数也是如此。）

生成的执行上下文会被压入执行上下文栈中，在处于栈顶的上下文执行完毕后，就会自动出栈。全局上下文在浏览器窗口关闭后出栈。

每个执行上下文都有三个重要属性：

- 变量对象（Variable Object）
- 作用域链（Scope chain）
- this

## 变量对象（Variable Object）与活动对象（Activation Object）

变量对象是与执行上下文相关的数据作用域，储存了在上下文中定义的变量和函数声明。

<!-- more -->

### 全局上下文的变量对象

全局上下文的变量对象就是全局对象。全局对象的特征：

- 可以通过this引用，在浏览器JavaScript中，全局对象就是Window对象；
`console.log(this) // Window`
- 全局对象是由Object构造函数实例化的一个对象；
`console.log(this instanceof Object) // true`
- 预定义了许多函数和属性
- 作为全局变量的宿主

```
var a = 1;

console.log(this.a) // 1
``` 

- 浏览器JavaScript中，全局对象有window属性指向自身。

### 函数上下文的变量对象

当调用一个函数时，一个新的执行上下文就会被创建。一个执行上下文的周期可以分为两个阶段，创建阶段和代码执行阶段。

- 创建阶段
在这个阶段，执行上下文会分别创建变量对象，建立作用域链，以及确定this的指向。
- 代码执行阶段
创建完之后，就会开始执行代码，这个时候会完成变量赋值，函数引用，以及执行其他代码。

#### 创建阶段，变量对象的创建

1. 创建变量对象，初始化有arguments一个属性。检查当前上下文中的参数，建立该对象下的属性与属性值。在变量对象中以形参名建立属性，属性值为实参。如果没有传入实参，则属性名为undefined。
2. 检查当前上下文的函数声明，也就是使用function关键字声明的函数。在变量对象中以函数名建立属性，属性值为指向该函数所在内存地址的引用。如果函数名属性已经存在，那么该属性会被新的引用所覆盖。
3. 检查当前上下文的变量声明。每找到一个变量声明，就在变量对象中以变量名建立一个属性，属性值为undefined。如果该变量名的属性已经存在，为了防止同名的函数被修改为undefined，会直接跳过，原属性不会被修改。

```
function foo (a) {
    var b = 2
    function c () {}
    var d = function () {}
    b = 3
    var c = 4
}

foo(1)
```

在全局作用域中运行`foo()`时，foo()的执行上下文开始创建：
 
```
fooEC = {
    VO: {},
    scopeChain: {},
    this: {}
}

VO = {
    arguments: {
        0: undefined,
        length: 3
    },
    a: 1,
    b: undefined,
    c: 函数c的地址引用,
    d: undefined
}
```

#### 执行阶段，变量对象的赋值

在未进入执行阶段之前，变量对象中的属性都不能访问。但是在进入执行阶段之后，变量对象（VO）转变为活动对象（AO），里面的属性都能被访问了，然后开始进行执行阶段的操作。
    
```
VO -> AO

AO = {
    arguments: {
        0: 1,
        length: 1
    },
    a: 1,
    b: 3,
    c: 4,
    d: 函数d的地址引用,
}
```

### 变量对象相关例子

```    
function foo () {
    console.log(a)
    a = 1
}

foo()

function bar () {
    a = 1
    console.log(a)
}
```

第一段报错：Uncaught ReferenceError: a is not defined，第二段打印1。因为只有声明才能够在当前上下文中的变量对象中创建属性。

## 作用域与作用域链

### 作用域

作用域是程序源代码中定义变量的区域，规定了如何查找变量，也就是确定当前执行代码对变量的访问权限。

ES6之前只有全局作用域和函数作用域。

JavaScript采用词法作用域（lexical scoping），也就是静态作用域。

#### 词法（静态）作用域与动态作用域

因为采用词法作用域，函数的作用域在函数定义的时候就已经决定了。与词法作用域相对的是动态作用域，函数的作用域在函数调用的时候才决定。

    
```
var value = 1
function func1 () {
    console.log(value)
}

function func2 () {
    var value = 2
    func1()
}

func2()
```

在JavaScript（词法作用域）中，func1的作用域在定义时就已经决定了，因此作用域等同于自己的作用域加上定义环境的作用域，因此控制台输出1。

而在采用动态作用域的语言中，func1的作用域在函数调用时才决定，作用域链等同于自己的作用域加上执行环境的作用域，因此控制台输出2。

### 作用域链

当查找变量时，会先从当前上下文的变量对象中查找。如果没有找到，就会从上层执行上下文的变量对象查找，一直找到全局上下文的变量对象，也就是全局对象。这样由多个执行上下文的变量对象构成的链表叫做作用域链。

#### 作用域链的构成

函数有一个内部属性[[scope]]，当函数创建的时候，就会保存所有父变量对象到其中，可以理解为[[scope]]就是所有父变量对象的层级链。
    
```
function foo () {
    function bar () {
        console.log(1)
    }
}

foo.[[scope]] = {
    globalContext.VO
}

bar.[[scope]] = {
    globalContext.VO,
    fooContext.AO
}
```

当函数激活时，进入函数上下文，创建VO后，就会将VO添加到作用域链的前端。（VO在执行上下文进入执行阶段时就变成了AO。）这时，执行上下文的作用域链为`[AO].concat([[scope]])`

### 闭包

在理解作用域链后，对闭包的理解也就更加深入了。

```
var fn1 = function () {
    var a = 1
    return function () {
        console.log(a)
    }
}

var fn2 = fn1()

fn2() // 控制台打印1
```

以上代码，外部函数fn1执行完毕后，内部函数匿名函数仍可以访问到fn1的变量。其本质原因是，返回的匿名函数，其作用域在创建时已经确定，包含着fn1的变量对象。通过`var fn2 = fn1()`，将该匿名函数赋值给fn2，仍保存着对fn1变量对象的引用，因此fn1的变量对象没有被垃圾回收，仍旧可以访问到。

### 总结函数执行上下文中作用域链与变量对象的创建过程

```
var scope = 'global scope'

function checkscope() {
    var scope2 = 'local scope'
    return scope2
}

checkscope()
```

1. 创建函数checkscope，保存父级变量对象到函数的[[scope]]属性；
2. 执行checkscope函数，生成checkscope函数执行上下文，将checkscope函数执行上下文压入执行上下文栈；
3. 执行上下文创建阶段，复制函数[[scope]]属性创建作用域链；
4. 创建变量对象，依次加入形参、函数声明、变量声明；
5. 将变量对象添加到作用域链前端，建立完整作用域链；
6. 执行上下文执行阶段，变量对象转变为活动对象，开始执行函数，随着函数的执行修改活动对象的属性值。 

### 作用域与执行上下文的区别

有了以上铺垫，也能看出来作用域与执行上下文是完全不同的东西。就我个人的理解，本质区别首先在于，执行上下文是有着变量对象、作用域链、this等重要属性的一个环境，而作用域，仅仅是作用域链的实现规则，也就是执行上下文中一个属性的实现规则。

其次：

> JavaScript代码的整个执行过程，分为两个阶段，代码编译阶段和代码执行阶段。编译阶段由编译器完成，将代码翻译成可执行代码，这个阶段作用域规则会确定。执行阶段由引擎完成，主要任务是执行可执行代码，执行上下文在这个阶段创建。

作用域与执行上下文的创建时间也不同。作用域规则在代码编译阶段被确定，表现在代码中，就如词法作用域时例子所示，函数在定义时作用域已经被确定，无论函数在哪里被调用，其作用域都不会改变。而执行上下文是在函数被调用时创建的，表现在代码中，就可以看到，即使是引用相同的函数，根据调用函数的方式不同，其中执行上下文的属性，如this（如fn()与window.fn()），也会不同。

会有人把执行上下文和作用域混淆，大概是因为，在执行上下文中，能否取到变量是通过作用域链来判断的，再加上定义上都可以说是一种环境（通俗来说，相当于是一个“大环境”，代码运行环境，一个“大环境”中的“小环境”，能取到的变量的环境。“大环境”会在函数每次被调用时创建，然后将“小环境”加入其中；“小环境”定义时已经确定，不会改变。当然，说是不会改变，只是能否取得变量不会改变，变量的值，如传入的参数等，当然会改变。这应该也是说作用域链是作用域的具体实现的原因，在作用域链具体实现时，当然就带上了实际的数值。），所以就有人想当然地把执行上下文与作用域混淆了。

## this

- 在全局上下文中，this指向全局对象，如在浏览器中，this指向window。
- 在函数上下文中，this由调用者提供，由调用函数的方式决定。
- 如果调用者函数被一个对象所拥有，那么该函数在调用时，this指向该对象；
- 如果函数独立调用，那么该函数内部的this指向undefined；
- 通过更改调用方式，即使用call或apply，可以指定this的指向。

- 在非严格模式下，当this指向undefined时，它会被自动指向全局对象。

### 关于“独立调用”

```
function fn () {
    'use strict'
    console.log(this)
}

fn() // fn是调用者，独立调用，this为undefined

window.fn() // fn为调用者，非独立调用，this为window
```
    
```
var foo = {
    fn: function () {
        return this
    }
}

var test = foo.fn
console.log(foo.fn()) // foo
console.log(test()) // window
```

因此，“独立调用”的定义非常明确，就是函数在被调用时是否有对象。哪怕函数引用相同，甚至只是本就在全局上下文中的fn与window.fn的区别，都是独立调用与非独立调用的例子。

### 使用call/apply显示指定this
 
```
var obj = {}
var fn = function () {
    console.log(this)
}

fn.call(obj) // obj
```

call与apply功能相同，唯一不同的是，call的参数一个一个传递（fn.call(obj, 10, 20)），apply的参数则以数组的形式传递（fn.apply(obj, [10, 20])）。

### 使用this实现继承

```
var Person = function (name, age) {
    this.name = name
    this.age = age
}

var Student = function (name, age, high) {
    Person.call(this, name, age)
    this.high = high
}

var student = new Student('Wang Yi', 'forever 21', '169')
```

在使用new操作符时，会进行以下几步：

1. 创建一个新对象；
2. 将构造函数中的this指向这个新对象；
3. 执行构造函数中的代码（为这个新对象添加属性）；
4. 返回新对象。

因此，以上步骤相当于将Person构造函数的this指向Student构造函数的实例，因此实现了继承。

### Nomal function 会自动创建上下文，而arrow function会从外部获取一个上下文。

## 参考文献

- 波同学，[前端基础进阶系列](http://www.jianshu.com/p/cd3fee40ef59)
- 冴羽，[JavaScript深入系列](https://github.com/mqyqingfeng/Blog)
