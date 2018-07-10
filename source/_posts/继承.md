---
title: JavaScript面向对象的程序设计-继承
date: 2017-04-16 15:56:47
categories: [JavaScript, 基础]
tags: [JavaScript]
---

## 原型链

将父构造函数的实例赋给子构造函数指向的原型对象，使得子构造函数指向的原型对象中有[[prototype]]指向父构造函数的原型对象。这样层层连接，构成原型链。

```
function SuperType () {
  this.property = 'super'
}
SuperType.prototype.getSuperProperty = function () {
  return this.property
}

function SubType () {
  this.subProperty = 'sub'
}

SubType.prototype = new SuperType()

SubType.prototype.getSubProperty = function () {
  return this.subProperty
}

var instance = new SubType()
console.log(instance.getSubProperty())
console.log(instance.getSuperProperty())
```

这样，新原型不仅有SuperType实例所具有的所有属性和方法，其内部还有一个指针，指向了SuperType的原型对象。最终结果就是：instance的[[prototype]]和SubType的prototype指向SubType的原型，SubType的原型（SuperType的实例）以及其他SuperType的实例的[[prototype]]，和SuperType的prototype指向SuperType的原型。再加上最上层prototype指向没有被更改的函数（此处为SuperType），它的原型为Object的实例。最终的原型链如图所示：

![原型链](http://wx4.sinaimg.cn/large/7b1152ffly1feot7ymu7vj212g0uwq9b.jpg)

### 判断实例与原型的关系

- 
instanceof。只要用这个操作符测试实例与原型链中出现过的构造函数，就会返回true。

```
instance instanceof Object // true
instance instanceof SubType // true
instance instanceof SuperType // true
```

- isPrototypeOf。只要是原型链中出现过的原型，就会返回true。

```
Object.prototype.isPrototypeOf(instance) // true
SubType.prototype.isPrototypeOf(instance) // true
SuperType.prototype.isPrototypeOf(instance) // true
```

### 原型链的问题

1. 只要是原型属性，都是所有实例共享同一个引用的。而利用原型链实现继承，子构造函数的原型对象就是父构造函数的一个实例，因而从父构造函数继承而来的其实例的实例属性也会变成子构造函数实例的原型属性，同样是多个实例共享一个（父构造函数那一个实例的）属性。（但是如果不是引用类型的属性，则不会造成困扰，因为在实例上直接赋值的属性，会生成实例属性覆盖原型属性。可是如果对引用类型使用push等，相当于直接操作原型属性，则会影响到所有的实例。）
2. 在创建子类型的实例时，不能在不影响所有对象实例的情况下，向超类型的构造函数中传递参数。

因此，实践中很少单独使用原型链。

## 借用构造函数（伪造对象/经典继承）

在子构造函数内部调用父构造函数，this指向子构造函数的实例，相当于借父构造函数用一用，然后再执行自己的代码。这样父构造函数定义的属性就真的成为了子构造函数实例的实例属性，而不是原型属性。

```
function SuperType () {
  this.colors = ['red', 'blue', 'green']
}

function SubType () {
  SuperType.call(this)
}

var instance1 = new SubType()
instance1.colors.push('black')
console.log(instance1.colors) // ['red', 'blue', 'green', 'black']

var instance2 = new SubType()
console.log(instance2.colors) // ['red', 'blue', 'green']
```

### 传递参数

```
function SuperType (colors) {
  this.colors = colors
}

function SubType () {
  SuperType.apply(this, arguments)
  this.color = 'pink'
}

var instance1 = new SubType(['red', 'blue', 'green', 'black'])
console.log(instance1.colors) // ['red', 'blue', 'green', 'black']

var instance2 = new SubType(['red', 'blue', 'green'])
console.log(instance2.colors) // ['red', 'blue', 'green']
console.log(instance2.color) // pink
```

### 借用构造函数的问题

如果仅仅使用借用构造函数，就无法避免构造函数模式存在的问题：

- 方法都在构造函数中定义，函数复用就无从谈起了。
- 在超类型的原型中定义的方法，子类型也是不可见的，结果所有类型都只能使用构造函数模式。

因此，借用构造函数技术也很少单独使用。

## 组合继承（伪经典继承）

将原型链和借用构造函数技术结合。使用原型链实现对原型属性和方法的继承，通过借用构造函数实现对实例属性的继承。

```
function SuperType (colors, color) {
  this.colors = colors
  this.color = color
}

SuperType.prototype = {
  getColor: function () {
    return this.color
  },
  constructor: SuperType
}

function SubType (colors, color, name) {
  SuperType.call(this, colors, color)
  this.name = name
}

SubType.prototype = new SuperType()
var instance1 = new SubType(['red', 'blue'], 'black', 'color1')
var instance2 = new SubType(['green'], 'green', 'color2')
console.log(instance1.getColor()) // black
console.log(instance2.getColor()) // green
console.log(instance1 === instance2)
```

子构造函数实例的实例属性在子构造函数中定义，方法在原型中定义。继承父构造函数的实例属性依靠借用构造函数模式，继承父构造函数原型对象的属性和方法依靠原型链。

### 组合继承的问题

重复调用两次超类型构造函数。一次是在给SubType.prototype赋值的时候，另一次是在SubType内部调用的时候。因此会导致在SubType所指向的原型对象中有一组属性（即超类型构造函数的实例属性。直接调用new SuperType()赋给SubType的prototype，目的只为继承SuperType的原型属性，在这个过程中，作为实例属性的属性算是副产物，为undefined），而在实例中也有一组属性（真的被用到的），屏蔽了原型中的那组undefined的属性。

## 原型式继承

借助原型基于已有的对象创建新对象，而不必因此创建自定义类型。

```
function object (o) {
  function F () {}
  F.prototype = o
  return new F()
}

var person = {
  name: 'Wang Yi',
  friends: ['Jacky', 'Shelby']
}

var anotherPerson = object(person)
anotherPerson.name = 'Greg'
anotherPerson.friends.push('Rob')

var yetAnotherPerson = object(person)
yetAnotherPerson.name = 'Linda'
yetAnotherPerson.friends.push('Barbie')
console.log(person.friends) // ["Jacky", "Shelby", "Rob", "Barbie"]
```

原型式继承必须有一个对象作为另一个对象的基础。相当于把一个对象作为新创建对象的原型。

没有必要兴师动众地创建构造函数，而只想让一个对象和另一个对象保持类似的情况下，原型式继承完全可以胜任。不过，这种方法与原型模式一样，包含引用类型值的属性始终会共享相应的值。

### Object.create()

ES5通过新增Object.create()方法规范了原型式继承。该方法接收两个参数，一个用作新对象原型的对象和（可选的）一个为新对象定义额外属性的对象。

Object.create()方法的第二个参数与Object.defineProperties()方法的第二个参数格式相同：每个属性都是通过自己的描述符定义的。

```
var anotherPerson = Object.create(person, {
  name: {
    value: 'Wang Yi'
  }
})
```

## 寄生式继承

简单说，就是在利用原型式继承创建了新对象之后，再利用工厂模式/寄生构造函数模式（两个方式本来就基本相同）给新对象添加属性或方法。

```
function object (o) {
  function F () {}
  F.prototype = o
  return new F()
}

function createPerson (original) {
  var clone = object(original)
  clone.sayHi = function () {
    console.log('Hi!')
  }
  return clone
}

var person = {
  name: 'Wang Yi',
  friends: ['Jacky', 'Shelby']
}

var anotherPerson = createPerson(person)
anotherPerson.sayHi()
```

在主要考虑对象而不是自定义类型或构造函数的情况下，寄生式继承也是一种有用的方式。

## 寄生组合式继承

寄生组合式继承可以解决组合继承的问题。寄生组合式继承，即通过借用构造函数来继承属性，通过原型链的混成方式来继承方法。其背后的基本思路是：不必为了指定子类型的原型而调用超类型的构造函数，我们所需要的无非就是超类型原型的一个副本而已。本质上，就是使用寄生式继承来继承超类型的原型，然后再将结果指定给子类型的原型。

```
function object (o) {
  function F () {}
  F.prototype = o
  return new F()
}

function inheritPrototype (subType, superType) {
  var prototype = object(superType.prototype)
  prototype.constructor = subType
  subType.prototype = prototype
}

function SuperType (name) {
  this.name = name
  this.colors = ['red', 'blue', 'green']
}

SuperType.prototype.sayName = function () {
  console.log(this.name)
}

function SubType (name, age) {
  SuperType.call(this, name)
  this.age = age
}

inheritPrototype(SubType, SuperType)

SubType.prototype.sayAge = function () {
  console.log(this.age)
}

var instance = new SubType('Wang Yi', 19)
```

这种模式的高效率在于不会重复调用SuperType()构造函数，而是用一个临时的函数指向SuperType的原型对象，再调用创建新实例，这样不会在SubType.prototype上创建不必要的、多余的属性。与此同时，原型链还能保持不变，因此，还能正常使用instanceof和isPrototypeOf()。

开发人员普遍认为寄生组合式继承是引用类型最理想的继承方式。
