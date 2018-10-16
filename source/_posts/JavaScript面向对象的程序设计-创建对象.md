---
title: JavaScript面向对象的程序设计-创建对象
date: 2017-04-15 13:56:26
categories: [JavaScript, 基础]
tags: [JavaScript]
---

虽然Object构造函数或对象字面量都可以用来创建单个对象，但这些方式有个明显缺点：使用同一个接口创建很多对象，会产生大量的重复代码。考虑到在ECMAScript中无法创建类，开发人员发明了一种函数，用函数来封装以特定接口创建对象的细节。

## 工厂模式

```
function createPerson (name, age, job) {
  var o = new Object()
  o.name = name
  o.age = age
  o.job = job
  o.getName = function () {
    console.log(this.name)
  }
  return o
}

var person1 = createPerson('Wang Yi', 'forever 21', 'FE')
var person2 = createPerson('Imaging boyfriend', '25', 'charming man')
```

工厂模式解决了创建多个相似对象的问题，但没有解决对象识别的问题（即怎样知道一个对象的类型）。

<!-- more -->

## 构造函数模式

```
function Person (name, age, job) {
  this.name = name
  this.age = age
  this.job = job
  this.getName = function () {
    alert(this.name)
  }
  return this
}

var person1 = new Person('Wang Yi', 'forever 21', 'FE')
var person2 = new Person('Imaging boyfriend', '25', 'charming man')
```

使用new操作符调用构造函数经历以下步骤：

1. 创建一个新对象；
2. 将构造函数中的this指向新对象；
3. 执行构造函数中的代码（为这个新对象添加属性）；
4. 返回新对象。

在前面例子的最后，person1和person2分别保存着Person的不同实例。这两个对象都有一个constructor属性，该属性指向Person。**记得回来补充关于prototype的更准确的说法**

```
person1.constructor === Person // true
person2.constructor === Person // true
person1.constructor === Object // false

// 检测对象类型使用instanceof更靠谱
person1 instanceof Person // true
person2 instanceof Person // true
person1 instanceof Object // true
```

创建自定义的构造函数意味着可以将它的实例标识为一种特定的类型，这正是构造函数胜过工厂模式的地方。在这个例子中，person1和person2之所以同时是Object的实例，是因为所有对象均继承自Object。**详细以后再记录**

构造函数也有缺点，就是每个方法都需要在实例上重新创建一遍。创建两个完成同样任务的Function实例没有必要，有this对象在，也不需要在执行代码前就把函数绑定在特定对象上。因此，可以将函数定义转移到构造函数外部来解决这个问题。

    
```
function Person () {
  this.name = 'Wang Yi'
  this.getName = getName
}

function getName () {
  /* 值得注意的是，原本根据函数作用域在定义时已经确定的事实
  ** 在这个全局getName函数中是取不到Person构造函数中的name的
  ** 但是getName函数执行上下文中的this又与调用方式有关
  ** 调用方式为new Person().getName()
  ** 则被调用的getName函数的this则指向new出的Person实例
  ** 即使它和全局的getName函数是同一个引用也是如此
  ** 因为this指向只与函数的调用方式有关
  ** 作用域问题不再存在 */
  console.log(this.name)
}

new Person().getName()
```

这样，实例就共享了全局作用域定义的同一个getName函数。这样的确解决了两个函数做同一件事的问题，但也产生了新问题，一是在全局作用域中定义的函数实际上只能被某个对象调用，折让全局函数名不副实，二是，如果对象需要定义许多方法，就需要在全局定义许多函数，于是我们这个自定义的引用类型就毫无封装性可言了。

## 原型模式

我们创建的每个函数都有一个prototype属性，这个属性是一个指针，指向一个对象，这个对象的用途是包含可以由特定类型的所有实例共享的属性和方法。如果按照字面意思来理解，prototype就是通过调用构造函数而创建的那个对象实例的原型对象。使用原型对象的好处是可以让所有对象实例共享它所包含的属性和方法。

```
function Person () {}
Person.prototype.name = 'Wang Yi'
Person.prototype.age = 'forever 21'
Person.prototype.job = 'FE'
Person.prototype.getName = function () {
    console.log(this.name)
}

var person1 = new Person()
var person2 = new Person()
person1.getName === person2.getName
```

但所有实例共享一个同一组属性和方法。

### 原型对象

只要创建了一个函数，就会为该函数创建一个prototype属性，这个属性指向函数的原型对象。在默认情况下，所有原型对象都会自动获得一个constructor属性，这个属性包含一个指向prototype属性所在函数的指针，也就是Person.prototype.constructor === Person。

创建了自定义构造函数后，其原型对象默认只有constructor方法，其他属性和方法则是从Object继承而来的。当调用构造函数创建一个实例后，该实例的内部将包含一个指针，指向构造函数的原型对象。ECMA-262第5版称这个指针为[[prototype]]。需要注意的是，这个指针指向的是构造函数的原型对象（构造函数也只是有一个属性prototype指向它而已），而不是构造函数，与构造函数没有直接的关系。

#### api-判断实例与原型对象的关系

- isPrototypeOf。Person.prototype.isPrototypeOf(person1)，如果对象内部有指向Person.prototype的指针，则返回true；
- getPrototypeOf。Object.getPrototypeOf(person1)，用于取得【实例】所指向的原型对象（而不是构造函数所指向的原型对象）。

#### 属性的检索顺序

读取某对象的某个属性时，先从对象实例本身的属性开始，如果找到了具有给定名字的属性，则返回该属性值。如果没有，则继续搜索指针指向的原型对象。即使将同名的实例属性设置为null，也不会去搜索原型对象。使用delete操作符才能够完全删除实例属性，从而能访问原型中的属性。

#### api-判断属性是否存在于对象

- hasOwnProperty。person1.hasOwnProperty(‘name’)，检测属性是否存在于实例中。即若name为person1的实例属性，返回true，若name为person1的原型属性或不存在此属性，返回false。
- in。有两种方式使用in操作符，单独使用或在for-in中使用。单独使用时，in操作符会在通过对象能够访问给定属性时返回true，无论该属性存在于实例中还是原型中。如：’name’ in person1，无论name存在于实例中还是原型中，都会返回true。
- 同时使用hasOwnProperty和in，就可以确定属性是存在于对象中还是原型中。

#### api-获得对象的属性

- for-in，返回所有能够通过对象访问的、【可枚举的】【属性】，其中既包括存在于实例中的属性，也包括存在于原型中的属性；
- Object.keys()方法，取得对象上所有【可枚举的】【实例属性】。接收一个对象作为参数，返回包含所有可枚举实例属性的字符串数组。顺序与for-in循环中出现的顺序一样。
- Object.getOwnPropertyNames，取得对象上所有【无论是否可枚举的】【实例属性】，返回包含所有实例属性的字符串数组。

#### 更简单的原型语法

每赋一个属性都要写一次Person.prototype太冗余，可以直接写一个对象，赋给Person.prototype。但这样Person.prototype.constructor不再指向Person，而是单纯地作为Object的实例，其中[[prototype]]中的constructor指向Object。因此需要重新赋值，Person.prototype = { constructor: Person }，且使用Object.defineProperty()将constructor属性设置为不可枚举。

另外，对象实例的[[prototype]]也是指向原本的原型对象的，在将构造函数的prototype指针指向新的原型对象后，原本创建的实例仍旧指向旧的原型对象。

### 原型模式的问题

所有实例共享同一套属性和方法，通常共享同一套属性不符合需求。

## 组合使用构造函数模式和原型模式

构造函数模式用于定义属性，原型模式用于定义方法和共享的属性。

构造函数与原型混成的模式，是目前在ECMAScript中使用最广泛、认同度最高的一种创建自定义类型的方法。可以说，这是用来定义引用类型的一种默认模式。

```
function Person (name, age, job) {
  this.name = name
  this.age = age
  this.job = job
  this.friends = ['Shelby', 'Count']
}

Person.prototype = {
  constructor: Person,
  getName: function () {
    console.log(this.name)
  }
}

var person1 = new Person('Wang Yi', 'forever 21', 'FE')
var person2 = new Person('Wang Yi\'s boyfriend in future', '25', 'FE')
console.log(person1.friends === person2.friends) // false
person1.friends.push('Van')
console.log(person1.friends) // ['Shelby', 'Count', 'Van']
console.log(person2.friends) // ['Shelby', 'Count']
console.log(person1.getName === person2.getName) // true
```

## 动态原型模式

为靠近其他OO语言的形式，将所有信息都封装在构造函数中。而通过在构造函数中初始化原型（仅在必要的情况下），又保持了同时使用构造函数和原型的优点。可以通过检测某个应该存在的方法是否有效，来决定是否需要初始化原型。

```
function Person (name, age, job) {
  this.name = name
  this.age = age
  this.job = job
  if (typeof this.getName != 'function') {
    Person.prototype.getName = function () {
      console.log(this.name)
    }
  }
}
```

在使用动态原型模式时，不能够使用对象字面量重写原型。因为第一次初始化原型时，已经有实例被创建，如果重写原型对象，则会切断已经创建的实例与原型对象的关系。

## 寄生构造函数模式

```
function Person (name, age) {
  var o = new Object()
  o.name = name
  o.age = age
  o.getName = function () {
    console.log(this.name)
  }
  return o
}

var friend = new Person('Wang Yi', 21)
friend.getName()
```

除了使用new操作符并把使用的包装函数成为构造函数外，这个模式和工厂模式没有区别。构造函数在不返回值的情况下，默认会返回新对象实例。而通过在构造函数末尾添加一个return语句，可以重写调用构造函数时返回的值。

假设我们想要创建具有额外方法的特殊数组，又不能直接修改Array的构造函数，就可以使用这个模式：

```
function SpecialArray () {
  var values = new Array()
  values.push.apply(values, arguments)
  values.toPipedString = function () {
    return this.join('|')
  }
  return values
}

var colors = new SpecialArray('red', 'blue', 'green')
console.log(colors.toPipedString()) // red|blue|green
```

寄生构造函数模式，返回的对象与构造函数SpecialArray的原型对象没有关系，也就是说，构造函数返回的对象与在函数外面创建没有什么不同。不能依赖instanceof操作符来确定对象类型。

## 稳妥构造函数模式

所谓稳妥对象，指的是没有公共属性，而且其方法也不引用this的对象。稳妥对象最适合在一些安全的环境中（这些环境会禁止使用this和new），或者在防止数据被其他应用程序改动时使用。稳妥构造函数遵循与寄生构造函数类似的模式，但有两点不同：一是新创建对象的实例方法不引用this；二是不使用new操作符调用构造函数。

```
function Person (name, age, job) {
  var o = new Object()
  o.getName = function () {
    console.log(name)
  }
  return o
}

var friend = Person('Wang Yi', 'forever 21', 'FE')
friend.getName() // 'Wang Yi'
```

这样，变量friend中保存的是一个稳妥对象，除了调用getName方法，没有别的方式可以访问其数据成员。

利用的就是闭包啦~

## 参考文献

- Nicholas C.Zakas，《JavaScript高级程序设计》
