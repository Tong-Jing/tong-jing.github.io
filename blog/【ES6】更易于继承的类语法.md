> 和其它面向对象编程语言一样，ES6 正式定义了 class 类以及 extend 继承语法糖，并且支持静态、派生、抽象、迭代、单例等，而且根据 ES6 的新特性衍生出很多有趣的用法。

#一、类的基本定义
基本所有面向对象的语言都支持类的封装与继承，那什么是类？

类是面向对象程序设计的基础，包含**数据封装、数据操作以及传递消息的函数**。类的实例称为对象。

ES5 之前通过函数来模拟类的实现如下：

```JavaScript
// 构造函数
function Person(name) {
  this.name = name;
}
// 原型上的方法
Person.prototype.sayName = function(){
  console.log(this.name);
};
// new 一个实例
var friend = new Person("Jenny");

friend.sayName(); // Jenny
console.log(friend instanceof Person);   // true
console.log(friend instanceof Object);   // true
```
**总结来说，定义一个类的思路如下**：
- 1.需要构造函数封装数据
- 2.在原型上添加方法操作数据，
- 3.通过New创建实例
  ES6 使用`class`关键字定义一个类，这个类有特殊的方法名`[[Construct]]`定义构造函数，在 new 创建实例时调用的就是`[[Construct]]`，示例如下：

```JavaScript
/*ES6*/
// 等价于 let Person = class {
class Person {
  // 构造函数
  constructor(name) {
    this.name = name;
  }
  // 等价于Person.prototype.sayName
  sayName() {
    console.log(this.name);
  }
}

console.log(typeof Person);   // function
console.log(typeof Person.prototype.sayName);   // function

let friend = new Person("Jenny");

friend.sayName(); // Jenny
console.log(friend instanceof Person);   // true
console.log(friend instanceof Object);   // true
```
上面的例子中`class`定义的类与自定义的函数模拟类功能上貌似没什么不同，但本质上还有很大差异的：
- 函数声明可以被提升，但是class类声明与let类似，不能被提升；
- 类声明自动运行在严格模式下，“use strict”；
- 类中所有方法都是不可枚举的，enumerable 为 false。

# 二、更灵活的类
类和函数一样，是JavaScript的一等公民（可以传入函数、从函数返回、赋值），并且注意到类与对象字面量还有更多相似之处，这些特点可以扩展出类更灵活的定义与使用。
## 2.1 拥有访问器属性
对象的属性有数据属性和访问属性，类中也可以通过`get`、`set`关键字定义访问器属性：

```JavaScript
class Person {
  constructor(name) {
    this.name = name;
  }

  get value () {
    return this.name + this.age
  }
  set value (num) {
    this.age = num
  }
}

let friend = new Person("Jenny");
// 调用的是 setter
friend.value = 18
// 调用的是 getter
console.log(friend.value) // Jenny18
```

## 2.2 可计算的成员名称
类似 ES6 对象字面量扩展的可计算属性名称，类也可以**用[表达式]定义可计算成员名称**，包括类中的方法和访问器属性：

```JavaScript
let methodName = 'sayName'

class Person {
  constructor(name) {
    this.name = name;
  }

  [methodName + 'Default']() {
    console.log(this.name);
  }

  get [methodName]() {
    return this.name
  }

  set [methodName](str) {
    this.name = str
  }
}

let friend = new Person("Jenny");

// 方法
friend.sayNameDefault(); // Jenny
// 访问器属性
friend.sayName = 'lee'
console.log(friend.sayName) // lee
```

> 想进一步熟悉对象新特性可参考：[【ES6】对象的新功能与解构赋值][1]

## 2.3 定义默认迭代器
 ES6 中常用的集合对象（数组、Set/Map集合）和字符串都是可迭代对象，如果类是用来表示值这些可迭代对象的，那么定义一个默认迭代器会更有用。

ES6 通过给`Symbol.iterator`属性添加生成器的方式，定义默认迭代器：

```JavaScript
class Person {
  constructor(name) {
    this.name = name;
  }

  *[Symbol.iterator]() {
    for (let item of this.name){
      yield item
    }
  }
}

var abbrName = new Person(new Set(['j', 'j', 'e', 'e', 'n', 'y', 'y', 'y',]))
for (let x of abbrName) {
  console.log(x); // j e n y
}
console.log(...abbrName) // j e n y
```
定义默认迭代器后类的实例就可以使用`for-of`循环和展开运算符（...）等迭代功能。

> 对以上迭代器内容感到困惑的可参考：[【ES6】迭代器与可迭代对象][2]
## 2.4 作为参数的类
类作为"一等公民”可以当参数使用传入函数中，当然也可以从函数中返回：

```JavaScript
function createClass(className, val) {
  return new className(val)
}

let person = createClass(Person,'Jenny')
console.log(person) // Person { name: 'Jenny' }
console.log(typeof person) // object
```

## 2.5 创建单例
使用类语法创建单例的方式通过new立即调用**类表达式**：

```JavaScript
let singleton = new class {
  constructor(name) {
    this.name = name;
  }
}('Jenny')
 
console.log(singleton.name) // Jenny
```
这里先创建**匿名类表达式**，然后 new 调用这个类表达式，并通过小括号立即执行，这种类语法创建的单例不会在作用域中暴露类的引用。
# 三、类的继承
回顾 ES6 之前如何实现继承？常用方式是通过原型链、构造函数以及组合继承等方式。

ES6 的类使用熟悉的**`extends`关键字指定类继承的函数**，并且可以通过**`surpe()`方法访问父类的构造函数**。

例如继承一个 Person 的类：
```JavaScript
class Friend extends Person {
  constructor(name, phone){
    super(name)
    this.phone = phone
  }
}

let myfriend = new Friend('lee',2233)
console.log(myfriend) // Friend { name: 'lee', phone: 2233 }
```
Friend 继承了 Person，术语上称 Person 为**基类**，Friend 为**派生类**。

需要注意的是，`surpe()`只能在派生类中使用，它负责初始化 this，所以派生类使用 this 之前一定要用`surpe()`。


## 3.1 继承内建对象
ES6 的类继承可以继承内建对象（Array、Set、Map 等），继承后可以拥有基类的所有内建功能。例如：

```JavaScript
class MyArray extends Array {
}

let arr = new MyArray(1, 2, 3, 4),
  subarr = arr.slice(1, 3)

console.log(arr.length) // 4
console.log(arr instanceof MyArray) // true
console.log(arr instanceof Array) // true
console.log(subarr instanceof MyArray) // true
```
注意到上例中，不仅 arr 是派生类 MyArray 的实例，subarr 也是派生类 MyArray 的实例，**内建对象继承的实用之处是改变返回对象的类型**。

> 浏览器引擎背后是通过`[Symbol.species]`属性实现这一行为，它被用于返回函数的静态访问器属性，内建对象定义了`[Symbol.species]`属性的有 Array、ArrayBuffer、Set、Map、Promise、RegExp、Typed arrays。

## 3.2 继承表达式的类
目前`extends`可以继承类和内建对象，但更强大的功能从表达式导出类！

这个表达式要求**可以被解析为函数并具有`[[Construct]]`属性和原型**，示例如下：

```JavaScript
function Sup(val) {
  this.value = val
}

Sup.prototype.getVal = function () {
  return 'hello' + this.value
}

class Derived extends Sup {
  constructor(val) {
    super(val)
  }
}

let der = new Derived('world')
console.log(der) // Derived { value: 'world' }
console.log(der.getVal()) // helloworld
```

## 3.3 只能继承的抽象类
ES6 引入`new.target`元属性判断函数是否通过new关键字调用。类的构造函数也可以通过`new.target`确定类是如何被调用的。

可以通过`new.target`创建抽象类（不能实例化的类），例如：

```JavaScript
class Abstract  {
  constructor(){
    if(new.target === Abstract) {
      throw new Error('抽象类（不能直接实例化）')
    }
  }
}

class Instantiable extends Abstract {
  constructor() {
    super()
  }
}

// let abs = new Abstract() // Error: 抽象类（不能直接实例化）
 let abs = new Instantiable()
console.log(abs instanceof Abstract) // true
```
虽然不能直接使用 Abstract 抽象类创建实例，但是可以作为基类派生其它类。
# 四、类的静态成员
ES6 使用`static`关键字声明静态成员或方法。在类的方法或访问器属性前都可以使用`static`，唯一的限制是不能用于构造函数。

静态成员的作用是某些类成员的私有化，及不可在实例中访问，必须要直接在类上访问。

```JavaScript
class Person {
  constructor(name) {
    this.name = name;
  }

  static create(name) {
    return new Person(name);
  }
}

let beauty = Person.create("Jenny");
// beauty.create('lee') // TypeError
```
如果基类有静态成员，那这些静态成员在派生类也可以使用。

例如将上例的 Person 作为基类，派生出 Friend 类并使用基类的静态方法create( )：
```JavaScript
class Friend extends Person {
  constructor(name){
    super(name)
  }
}

var friend = Friend.create('lee')
console.log(friend instanceof Person) // true
console.log(friend instanceof Friend) // false
```
可以看出派生类依然可以使用基类的静态方法。

----------

*推荐阅读《深入理解ES6》*

加油哦少年！

[1]: https://segmentfault.com/a/1190000016746873
[2]: https://segmentfault.com/a/1190000016824284