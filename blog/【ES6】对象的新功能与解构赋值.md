> ES6 通过字面量语法扩展、新增方法、改进原型等多种方式加强对象的使用，并通过解构简化对象的数据提取过程。


# 一、字面量语法扩展

在 ES6 模式下使用字面量创建对象更加简洁，对于对象属性来说，**属性初始值可以简写**，并可以使用**可计算的属性名称**。对象方法的定义消除了冒号和 function 关键字，示例如下：

```JavaScript
// Demo1
var value = "name", age = 18
var person = {
  age, // age: age
  ['my' + value]: 'Jenny',  // myname
  sayName () {  // sayName: function()
    console.log(this.myname)
  }
}
console.log(person.age) // 18
console.log(person.myname) // Jenny
person.sayName(); // Jenny
```

针对重复定义的对象字面量属性，ES5严格模式下会进行**重复属性检查**从而抛出错误，而ES6移除了这个机制，无论严格模式还是非严格模式，**同名属性都会取最后一个值**。

```JavaScript
// demo2
var person = {
  ['my' + value]: 'Jenny',
  myname: 'Tom',
  myname: 'Lee',
}
console.log(person.myname) // Lee
```

# 二、新增方法
> 从 ES5 开始遵循的一个设计目标是，避免创建新的全局函数，也不在`object.prototype`上创建新的方法。
为了是某些任务更容易实现，ES6 在全局 Object 对象上引入一些新的方法。


## 2.1 Object.is( )

ES6 引入`Object.is()`方法来弥补全等运算符的不准确计算。

全等运算符在比较时不会触发强制转换类型，`Object.is()`运行结果也类似，但对于 +0 和 -0（在 JS 引擎中为两个不同实体）以及特殊值`NaN`的比较结果不同，示例来看：

```JavaScript
// demo3
console.log(5 == '5') // true
console.log(5 === '5') // false
console.log(Object.is(5, '5')) // false

console.log(+0 == -0) // true
console.log(+0 === -0) // true
console.log(Object.is(+0, -0)) // false

console.log(NaN == NaN) // false
console.log(NaN === NaN) // false
console.log(Object.is(NaN, NaN)) // true
```
总结来说，`Object.is()`对所有值进行了更严格等价判断。当然，是否使用`Object.is()`代替全等操作符（===）取决于这些特殊情况是否影响代码。


## 2.2 Object.assign( )
ES6 添加`Object.assign()`来实现混合（Mixin）模式，即一个对象**接收**另一个对象的属性和方法。注意是接收而不是继承，例如接收 demo1 中的对象：

```JavaScript
// demo4
var friend = {}
Object.assign(friend, person)
friend.sayName() // Jenny
console.log(friend.age) // 18
console.log(Object.getPrototypeOf(friend) === person) // false
```
在`Object.assign()`之前，许多 JS 库自定义了混合方法 mixin( ) 来实现对象组合，代码类似于：

```JavaScript
function mixin(receiver, supplier) {
  Object.keys(supplier).forEach(function (key) {
    receiver[key] = supplier[key]
  })
  return receiver
}
```
可以看出 mixin( ) 方法使用“=”赋值操作，并不能复制**访问器属性**，同理`Object.assign()`也不能复制访问器属性，只是执行了赋值操作，访问器属性最终会转变为接收对象的数据属性。示例如下：

```JavaScript
// demo5
var animal = {
  name: 'lili',
  get type () {
    return this.name + type
  },
  set type (news) {
    type = news
  }
}
animal.type = 'cat'
console.log(animal.type) // lilicat

var pet = {}
Object.assign(pet, animal)
console.log(animal) // { name: 'lili', type: [Getter/Setter] }
console.log(pet) // { name: 'lili', type: 'lilicat' }
```

## 2.3 Object.setPrototypeOf( )

正常情况下对通过**构造函数**或`Object.create()`创建时，原型是被指定的。ES6 添加`Object.setPrototypeOf()` 方法来改变对象的原型。

例如创建一个继承 person 对象的 coder 对象，然后改变 coder 对象的原型：

```JavaScript
// demo6
let person = {
  myname: 'Jenny',
  sayName () { 
    console.log(this.myname)
  }
}

// 创建原型为 person 的 coder 对象
let coder = Object.create(person) 
coder.sayName() // Jenny
console.log(Object.getPrototypeOf(coder) === person) // true

let hero = {
  myname: 'lee',
  sayName () {
    console.log(this.myname)
  }
}

// 改变 coder 对象的原型为 hero
Object.setPrototypeOf(coder, hero)
coder.sayName() // lee
console.log(Object.getPrototypeOf(coder) === hero) // true
```
对象原型被存储在内部专有属性[[Prototype]]，调用`Object.getPrototypeOf()`返回存储在其中的值，调用`Object.setPrototypeOf()`改变其值。这个方法加强了对对象原型的操作，下一节重点讲解其它操作原型的方式。


# 三、增强对象原型
原型是 JS 继承的基础，ES6 针对原型做了很多改进，目的是更灵活地方式使用原型。除了新增的`Object.setPrototypeOf()`改变原型外，还引入`Super`关键字简化对原型的访问。

## 3.1 Super关键字
ES6 引入`Super`来更便捷的访问对象原型，上一节介绍 ES5 可以使用`Object.getPrototypeOf()`返回对象原型。举例说明`Super`的便捷，当对象需要复用原型方法，重新定义自己的方法时，两种实现方式如下：

```
// demo7
let coder1 = {
  getName () {
    console.log("coder1 name: ")
    Object.getPrototypeOf(this).sayName.call(this)
  }
}

// 设置 coder1 对象的原型为 hero（demo6）
Object.setPrototypeOf(coder1, hero)
coder1.getName() // coder1 name: lee

let coder2 = {
  getName () {
    console.log("coder2 name: ")
    super.sayName()
  }
}

Object.setPrototypeOf(coder2, hero)
coder2.getName() // coder2 name: lee
```
在 coder1 对象的 getName 方法还需要`call(this)`保证使用的是原型方法的 this，比较复杂，并且在**多重继承会出现递归调用栈溢出错误**，而直接使用`Super`就很简单安全。

**注意必须在简写方法中使用`Super`，要不然会报错**，例如以下代码运行语法错误：

```
let coder4= {
  getName: function () { // getName () 正确
    super.sayName() // SyntaxError: 'super' keyword unexpected here
  }
```
因为在例子中 getName 成为了匿名 function 定义的属性，在当前上下问调用`Super`引用是非法的。如果不理解，可以进一步看下方法的从属对象。

## 3.2 方法的从属对象
ES6 之前“方法”是具有功能而非数据的对象属性，ES6 正式将方法定义为有 `[[HomeObject]]`内部属性的函数。

`[[HomeObject]]`属性存储当前方法的从属对象，例如：

```
let coder5 = {
  sayName () {
    console.log("I have HomeObject")
  }
}

function shareName () {
    console.log("No HomeObject")
}
```
coder5 对象的 sayName( ) 方法的`[[HomeObject]]`属性值为 coder5，而 function 定义的 shareName( ) 没有将其赋值给对象，所以没有定义其`[[HomeObject]]`属性，这在使用`Super`时很重要。

`Super`就是在`[[HomeObject]]`属性上调用`Object.getPrototypeOf()`获得原型的引用，然后搜索原型得到同名函数，最后设置 this 绑定调用相应方法。

# 四、解构赋值
ES6 为数组和对象字面量提供了新特性——解构，可以简化数据提取的过程，减少同质化的代码。解构的基本语法示例如下：

```Javascript
let user = {
  name: 'jenny',
  id: 18
}
let {name, id} = user
console.log(name, id) // jenny 18
```
注意在这段代码中，user.name 存储在与对象属性名同名的 name 变量中。
## 4.1 默认值
如果解构时变量名称与对象属性名不同，即在对象中不存在，那么这个变量会默认为`undefined`：

```Javascript
let user = {
  name: 'jenny',
  id: 18
}
let {name, id, job} = user
console.log(name, id, job) // jenny 18 undefined
```

## 4.2 非同名变量赋值
非同名变量的默认值为`undefined`，但更多时候是需要为其赋值的，并且会将对象属性值赋值给非同名变量。ES6 为此提供了扩展语法，与对象字面量属性初始化程序很像：

```Javascript
let user = {
  name: 'jenny',
  id: 18
}
let {name, id = 16, job = 'engineer'} = user
console.log(name, id, job) // jenny 18 engineer

let {name: localName, id: localId} = user
console.log(localName, localId) // jenny 18

let {name: otherName = 'lee', job: otherJob = 'teacher'} = user
console.log(otherName, otherJob) // jenny teacher
```        
                                                                                                                                                     
可以看出这种语法实际与对象字面量相反，**赋值名在冒号左，变量名在右**，并且解构赋值时，只是更新了默认值，不能覆盖对象原有的属性值。

                                                                                                                                                                                                                                                            
## 4.3 嵌套解构
解构嵌套对象的语法仍然类似对象字面量，使用**花括号**继续查找下层结构：

```Javascript
let user = {
  name: 'jenny',
  id: 18,
  desc: {
    pos: {
      lng: 111,
      lat: 333
    }
  }
}

let {desc: {pos}} = user
console.log(pos) // { lng: 111, lat: 333 }

let {desc: {pos: {lng}}} = user
console.log(lng) // 111

let {desc: {pos: {lng: longitude}}} = user
console.log(longitude) // 111
```

# 五、对象类别
ES6 规范定义了对象的类别，特别是针对浏览器这样的执行环境。

- **普通（Ordinary）对象**
    具有 JS 对象所有的默认内部行为
- **特异（Exotic）对象**
    具有某些与默认行为不符的内部行为
- **标准（Standard）对象**
    ES6 规范中定义的对象
    可以是普通对象或特异对象，例如 Date、Array 等
- **内建对象**
    脚本开始执行时存在于 JS 执行环境中的对象
    所有标准对象都是内建对象



----------

*推荐阅读《深入理解ES6》*

----------
#### 加油哦，少年~
