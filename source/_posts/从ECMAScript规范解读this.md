---
title: 从ECMAScript规范解读this
date: 2017-07-18 14:56:21
categories: [JavaScript, 基础]
tags: [JavaScript]
---

## Types

ECMAScript的类型分为语言类型和规范类型。

ECMAScript的语言规范是开发者可以直接使用ECMAScript操作的，就是我们通常所说的Undefined、Null、Number、Boolean、String、Object；

而规范类型是用来用算法描述ECMAScript语言结构和ECMAScript语言类型的。规范类型包括Reference、List、Completion、Property Descriptor、Property Identifier、Lexical Environment和Environment Record。

## Reference

Reference类型是用来解释诸如delete、typeof以及赋值等操作行为的。Reference是一个Specification Type，也就是“只存在于规范里的抽象类型”。它是为了更好地描述语言的底层行为逻辑才存在的，但并不存在于实际的JS代码中。

Reference由三个部分组成，分别是：

- base value
- referenced name
- strict reference

base value就是属性所在的对象或者就是EnvironmentRecord，它的值只可能是undefined、一个对象、一个布尔值、一个字符串、一个数字或者一个environment record其中一种；

referenced name就是属性的名称。

<!-- more -->

``` JavaScript
var foo = 1
// 对应的Reference是：
var fooReference = {
  base: EnvironmentRecord,
  name: 'foo',
  strict: false
}
```

``` JavaScript
var foo = {
  bar: function () {
    return this
  }
}

foo.bar() // foo

var barReference = {
  base: foo,
  propertyName: 'bar',
  strict: false
}
```

### GetBase

> GetBase(V). Returns the base value component of the reference V.

返回reference的base value

### IsPropertyReference

> IsPropertyReference(V). Returns true if either the base value is an object or HasPrimitiveBase(V) is true; otherwise returns false.

简单的理解，如果base value是一个对象，就返回true。

### GetValue

GetValue方法可以从Reference类型获取对应值。

``` JavaScript
var foo = 1
var fooReference = {
  base: EnvironmentRecord,
  name: 'foo',
  strict: false
}

GetValue(fooReference) // 1
```

GetValue返回对象属性真正的值，即调用GetValue，返回的将是具体的值，而不再是一个Reference。

## 如何确定this的值

规范11.2.3 Function Calls，这里讲述了当函数调用时，如何确定this的取值：

1. 计算MemberExpression的结果赋给ref；
2. 判断ref是不是一个Reference类型：
 2.1 如果ref是一个Reference类型，并且IsPropertyReference(ref)是true，那么this的值为GetBase(ref)；
 2.2 如果ref是一个Reference类型，并且base value的值是Environment Record，那么this的值为ImplicitThisValue(ref)；
 2.3 如果ref不是一个Reference类型，那么this的值为undefined。

### MemberExpression

``` JavaScript
function foo () {
  console.log(this)
}
foo() // MemberExpression是foo

function foo () {
  return funtion () {
    console.log(this)
  }
}
foo()() // MemberExpression是foo()

var foo = {
  bar: function () {
    return this
  }
}
foo.bar() // MemberExpression是foo.bar
```

简单理解，MemberExpression就是()左边的部分。

### 判断ref是不是一个Reference类型

第一步是“计算MemberExpression的值赋给ref”，因此ref是不是一个Reference类型，关键在于看规范如何处理各种MemberExpression，返回的结果是不是一个Reference类型。

``` JavaScript
var value = 1
var foo = {
  value: 2,
  bar: function () {
    return this.value
  }
}

console.log(foo.bar())
console.log((foo.bar)())
console.log((foo.bar = foo.bar)())
console.log((false || foo.bar)())
console.log((foo.bar, foo.bar)())
```

#### foo.bar()

MemberExpression的计算结果为foo.bar，首先确定foo.bar是不是一个Reference类型。

根据规范11.2 Property Accessors，展示了一个计算过程，最后一步：

> Return a value of type Reference whose base value is baseValue and whose referenced name is propertyNameString, and whose strict mode flag is strict.

因此，foo.bar是一个Reference类型，该值为：

``` JavaScript
var reference = {
  base: foo,
  name: 'bar',
  strict: false
}
```

按照2.1的判断流程：

> 2.1 如果ref是Reference类型，并且IsPropertyReference(ref)是true，那么this的值为GetBase(ref)。

base value为foo，是一个对象，因此IsPropertyReference返回true，那么this的值为GetBase(ref)，即foo。

#### (foo.bar)()

foo.bar被()包住，根据规范11.1.6 The Grouping Operator，直接查看结果部分：

> Return the result of evaluating Expression. This may be of type Reference.
> Note this algorithm does not apply GetValue to the result of evaluating Expression.

实际上()没有对MemberExpression进行GetValue计算，因此结果与示例一相同。

#### (foo.bar = foo.bar)

赋值操作符，根据规范11.13.1 Simple Assignment（=）：
计算的第三步：

> 1. Let rval be GetValue(rref).

因为使用了GetValue，所以返回值不是Reference类型。按照之前的逻辑：

> 2.3 如果ref不是一个Reference类型，那么this的值为undefined

this为undefined。在非严格模式下，this的值为undefined的时候，其值会被隐式替换为全局对象。

#### (false || foo.bar)()

逻辑与算法，根据规范11.11 Binary Logical Operators：

> 1. Let lval be GetValue(lref).

因为使用了GetValue，所以返回的不是Reference类型，this为undefined。

#### (foo.bar, foo.bar)

逗号操作符，根据规范11.14 Comma Operator（,）：

> 1. Call GetValue(lref).

因为使用了逗号操作符，所以返回的不是Reference类型，this为undefined。

#### foo()

``` JavaScript
function foo () {
  console.log(this)
}

foo()
```

MemberExpression是foo，解析标识符，根据规范10.3.1 Identifier Resolution，会返回一个Reference类型的值：


``` JavaScript
var fooReference = {
  base: EnvironmentRecord,
  name: 'foo',
  strict: false
}
```

接下来进行判断：

> 2.1 如果ref是一个Reference类型，并且IsPropertyReference(ref)是true，那么this的值为GetBase(ref)。

因为base value是EnvironmentRecord，并不是一个Object类型，IsPropertyReference(ref)为false，进行下一条判断：

> 2.2 如果ref是一个Reference类型，并且base value为EnvironmentRecord，那么this的值为ImplicitThisValue(ref)

base value的值正是EnvironmentRecord，因此调用implicitThisValue(ref)。根据规范10.2.1.1.6，该函数始终返回undefined，因此this的值为undefined。

## 原文

- 冴羽，[JavaScript深入之从ECMAScript规范解读this](https://github.com/mqyqingfeng/Blog/issues/7)
