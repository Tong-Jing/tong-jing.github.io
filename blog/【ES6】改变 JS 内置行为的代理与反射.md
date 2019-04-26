> 代理（Proxy）可以拦截并改变 JS 引擎的底层操作，如数据读取、属性定义修改、函数构造等一系列操作。ES6 通过对这些底层内置对象的代理陷阱和反射函数，让开发者能进一步接近 JS 引擎的能力。
# 一、代理与反射的基本概念
**什么是代理和反射呢？**
代理是用来替代另一个对象（target），JS 通过`new Proxy()`创建一个目标对象的代理，该代理与该目标对象表面上**可以被当作同一个对象来对待**。


当目标对象上的进行一些特定的底层操作时，代理允许你**拦截这些操作并且覆写**它，而这原本只是 JS 引擎的内部能力。

> 如果你对些代理&反射的概念比较困惑的话，***可以直接看后面的应用示例***，最后再重新看这些定义就会更清晰！

拦截行为使用了一个能够响应特定操作的函数（ 被称为陷阱），每个代理陷阱对应一个反射（Reflect）方法。

ES6 的反射 API 以 `Reflect` 对象的形式出现，对象每个方法都与对应的陷阱函数同名，并且接收的参数也与之一致。以下是 `Reflect` 对象的一些方法：



代理陷阱          | 覆写的特性         | 方法       |
--------------------|------------------|-----------------------|
get | 读取一个属性的值| Reflect.get()
set| 写入一个属性 |Reflect.set()
has| in 运算符 | Reflect.has()
deleteProperty| delete 运算符 | Reflect.deleteProperty()
getPrototypeOf |Object.getPrototypeOf()| Reflect.getPrototypeOf()
isExtensible|Object.isExtensible()| Reflect.isExtensible()
defineProperty| Object.defineProperty()| Reflect.defineProperty
apply| 调用一个函数 |Reflect.apply()
construct |使用 new 调用一个函数| Reflect.construct()

每个陷阱函数都可以重写 JS 对象的一个特定内置行为，允许你拦截并修改它。

综合来说，想要控制或改变JS的一些底层操作，可以先创建一个***代理对象***，在这个代理对象上***挂载一些陷阱函数***，陷阱函数里面有***反射方法***。通过接下来的应用示例可以更清晰的明白代理的过程。

# 二、开始一个简单的代理
当你使用 Proxy 构造器来创建一个代理时，需要传递两个参数：目标对象（target）以及一个处理器（ handler），

先**创建一个仅进行传递的代理**如下：

```JavaScript
// 目标对象
let target = {}; 
// 代理对象
let proxy = new Proxy(target, {});

proxy.name = "hello";
console.log(proxy.name); // "hello"
console.log(target.name); // "hello"

target.name = "world";
console.log(proxy.name); // "world"
console.log(target.name); // "world
```

上例中的 proxy 代理对象将所有操作直接传递给 target 目标对象，**代理对象 proxy 自身并没有存储该属性，它只是简单将值传递给 target 对象**，proxy.name 与 target.name 的属性值总是相等，因为它们都指向 target.name。

此时代理陷阱的处理器为空对象，当然处理器可以定义了一个或多个陷阱函数。

## 2.1 set 验证对象属性的存储

假设你想要创建一个对象，并要求其属性值只能是数值，这就意味着该对象的每个新增属性
都要被验证，并且在属性值不为数值类型时应当抛出错误。

这时需要使用 set 陷阱函数来拦截传入的 value，该陷阱函数能接受四个参数：

-  trapTarget ：将接收属性的对象（ 即代理的目标对象）
- key ：需要写入的属性的键（ 字符串类型或符号类型）
- value ：将被写入属性的值；
- receiver ：操作发生的对象（ 通常是代理对象）

set 陷阱对应的反射方法和默认特性是`Reflect.set()`，和陷阱函数一样接受这四个参数，并会基于操作是否成功而返回相应的结果：

```
let targetObj = {};
let proxyObj = new Proxy(targetObj, {
  set: set
});

/* 定义 set 陷阱函数 */
function set (trapTarget, key, value, receiver) {
  if (isNaN(value)) {
     throw new TypeError("Property " + key + " must be a number.");
  }
  return Reflect.set(trapTarget, key, value, receiver);
}

/* 测试 */
proxyObj.count = 123;
console.log(proxyObj.count); // 123
console.log(targetObj.count); // 123

proxyObj.anotherName = "proxy" // TypeError: Property anotherName must be a number.
```

示例中set 陷阱函数成功拦截传入的 value 值，你可以尝试一下，如果**注释或不`return Reflect.set()`会发生什么？**，答案是拦截陷阱就不会有反射响应。

需要注意的是，直接给 targetObj 目标对象赋值时是不会触发 set 代理陷阱的，需要通过给代理对象赋值才会触发 set 代理陷阱与反射。


## 2.2 get 验证对象属性的读取
JS 非常有趣的特性之一，是读取不存在的属性时并不会抛出错误，而会把`undefined`当作该属性的值。

对于大型的代码库，当属性名称存在书写错误时（不会抛错）会导致严重的问题。这时**使用 get 代理陷阱验证对象结构（Object Shape)，访问不存在的属性时就抛出错误**，使对象结构验证变得简单。

get 陷阱函数会在读取属性时被调用，即使该属性在对象中并不存在，它能接受三个参数：

- trapTarget ：将会被读取属性的对象（ 即代理的目标对象） 
- key ：需要读取的属性的键（ 字符串类型或符号类型） 
- receiver ：操作发生的对象（ 通常是代理对象）

`Reflect.get()`方法接受与之相同的参数，并返回默认属性的默认值。

```
let proxyObj = new Proxy(targetObj, {
  set: set,
  get: get
});

/* 定义 get 陷阱函数 */
function get(trapTarget, key, receiver) {
  if (!(key in receiver)) {
    throw new TypeError("Property " + key + " doesn't exist.");
  }
  return Reflect.get(trapTarget, key, receiver);
}

console.log(proxyObj.count); // 123
console.log(proxyObj.newcount) // TypeError: Property newcount doesn't exist.
```

这段代码允许添加新的属性，并且此后可以正常读取该属性的值，但当读取的属性并
不存在时，程序抛出了一个错误，而不是将其默认为`undefined`。


#### 还可以使用 has 陷阱验证`in`运算符，使用 deleteProperty 陷阱函数避免属性被`delete`删除。

注：`in`运算符用于判断对象中是否存在某个属性，如果自有属性或原型属性匹配这个名称字符串或Symbol，那么`in`运算符返回 true。

```
targetObj = {
  name: 'targetObject'
};
console.log("name" in targetObj); // true
console.log("toString" in targetObj); // true
```
其中 name 是对象自身的属性，而 toString 则是原型属性（ 从 Object 对象上继承而来），所以检测结果都为 true。

has 陷阱函数会在使用`in`运算符时被调用，并且会传入两个参数（同名反射`Reflect.has()`方法也一样）：

- trapTarget ：需要读取属性的对象（ 代理的目标对象） 
- key ：需要检查的属性的键（ 字符串类型或 `Symbol`符号类型) 

deleteProperty 陷阱函数会在使用`delete`运算符去删除对象属性时下被调用，并且也会被传入两个参数（`Reflect.deleteProperty()` 方法也接受这两个参数）：

- trapTarget ：需要删除属性的对象（ 即代理的目标对象） ；
- key ：需要删除的属性的键（ 字符串类型或符号类型） 。

> **一些思考**：分析过 Vue 源码的都了解过，给一个 Vue 实例中挂载的 data，是通过`Object.defineProperty`代理 vm._data 中的对象属性，实现双向绑定...... 同理可以考虑使用 ES6 的 Proxy 的 get 和 set 陷阱实现这个代理。

# 三、对象属性陷阱
## 3.1 数据属性与访问器属性

ES5 最重要的特征之一就是引入了 `Object.defineProperty() `方法定义属性的特性。属性的特性是为了实现javascript引擎用的，属于内部值，因此不能直接访问他们。

属性分为数据属性和访问器属性。使用`Object.defineProperty()`方法修改数据属性的特性值的示例如下：

```
let obj1 = {
  name: 'myobj',
}
/* 数据属性*/
Object.defineProperty(obj1,'name',{
  configurable: false, // default true
  writable: false,     // default true
  enumerable: true,    // default true
  value: 'jenny'       // default undefined
})
console.log(obj1.name) // 'jenny'
```
其中`[[Configurable]] `表示能否通过 delete 删除属性从而重新定义为访问器属性；`[[Enumerable]] `表示能否通过`for-in`循环返回属性；`[[Writable]] `表示能否修改属性的值； `[[Value]] ` 包含这个属性的数据值。



对于访问器属性，该属性不包含数据值，包含一对getter和setter函数，定义访问器属性必须使用`Object.defineProperty()`方法：

```
let obj2 = {
  age: 18
}
/* 访问器属性 */
Object.defineProperty(obj2,'_age',{
  configurable: false, // default true
  enumerable: false,   // default true
  get () {             // default undefined
    return this.age
  },
  set (num) {          // default undefined
    this.age = num
  }
})
/* 修改访问器属性调用 getter */
obj2._age = 20  
console.log(obj2.age)  // 20

/* 输出访问器属性 */
console.log(Object.getOwnPropertyDescriptor(obj2,'_age')) 
// { get: [Function: get],
//   set: [Function: set],
//   enumerable: false,
//   configurable: false }

```

`[[Get]] `在读取属性时调用的函数， `[[Set]] ` 再写入属性时调用的函数。使用访问器属性的常用方式，是设置一个属性的值导致其他属性发生变化。


## 3.2 检查属性的修改

代理允许你使用 defineProperty 同名函数陷阱函数拦截`Object.defineProperty()`的调用，defineProperty 陷阱函数接受下列三个参数：

- trapTarget ：需要被定义属性的对象（ 即代理的目标对象）；
- key ：属性的键（ 字符串类型或符号类型）；
- descriptor ：为该属性准备的描述符对象。

defineProperty 陷阱函数要求在操作后返回一个布尔值用于判断操作是否成功，如果返回了 false 则抛出错误，故可以使用该功能来限制哪些属性可以被Object.defineProperty() 方法定义。

例如，如果想阻止定义`Symbol`符号类型的属性，你可以检查传入的属性值，若是则返回 false：

```
/* 定义代理 */
let proxy = new Proxy({}, {
  defineProperty(trapTarget, key, descriptor) {
    if (typeof key === "symbol") {
      return false;
    }
    return Reflect.defineProperty(trapTarget, key, descriptor);
  }
});

Object.defineProperty(proxy, "name", {
  value: "proxy"
});
console.log(proxy.name); // "proxy"

let nameSymbol = Symbol("name");
// 抛出错误
Object.defineProperty(proxy, nameSymbol, {
  value: "proxy"
})
```

# 四、函数代理

## 4.1 构造函数 & 立即执行

函数的两个内部方法：`[[Call]]` 与`[[Construct]]`会在函数被调用时调用，通过代理函数来为这两个内部方法设置陷阱，从而控制函数的行为。


**`[[Construct]]`会在函数被使用`new`运算符调用时执行**，代理触发`construct()`陷阱函数，并和`Reflect.construct()`一样接收到下列两个参数：

- trapTarget ：被执行的函数（ 即代理的目标对象） ；
- argumentsList ：被传递给函数的参数数组。

**`[[Call]]`会在函数被直接调用时执行**，代理触发`apply()`陷阱函数，它和`Reflect.apply()`都接收三个参数：

- trapTarget ：被执行的函数（ 代理的目标函数） ；
- thisArg ：调用过程中函数内部的 this 值；
- argumentsList ：被传递给函数的参数数组。

> 每个函数都包含`call()`和`apply()`方法，用于重置函数运行的作用域即 this 指向，区别只是接收参数的方式不同：`call()`的参数需要逐个列举、`apply()`是参数数组。

显然，*apply 与 construct 要求代理目标对象必须是一个函数，*这两个代理陷阱在函数的执行方式上开启了很多的可能性，结合使用就可以完全控制任意的代理目标函数的行为。

## 4.2 验证函数的参数

看到`apply()`和`construct()`陷阱的参数都有被传递给函数的参数数组`argumentsList`，所以可以用来验证函数的参数。

例如需要保证所有参数都是某个特定类型的，并且不能通过 new 构造使用，示例如下：

```
/* 定义 sum 目标函数 */
function sum(...values) {
  return values.reduce((previous, current) => previous + current, 0);
}
/* 定义 apply 陷阱函数 */
function applyRef (trapTarget, thisArg, argumentList) {
  argumentList.forEach((arg) => {
    if (typeof arg !== "number") {
      throw new TypeError("All arguments must be numbers.");
    }
  });
  return Reflect.apply(trapTarget, thisArg, argumentList);
}
/* 定义 construct 陷阱函数 */
function constructRef () {
  throw new TypeError("This function can't be called with new.");
}
/* 定义 sumProxy 代理函数 */
let sumProxy = new Proxy(sum, {
  apply: applyRef,
  construct: constructRef
});

console.log(sumProxy(1, 2, 3, 4)); // 10

// console.log(sumProxy(1, "2", 3, 4)); // TypeError: All arguments must be numbers.
// let result = new sumProxy() // TypeError: This function can't be called with new.

```

sum() 函数会将所有传递进来的参数值相加，此代码通过将 sum() 函数封装在 sumProxy() 代理中，如果传入参数的值不是数值类型，该函数仍然会尝试加法操作，但在函数运行之前拦截了函数调用，触发`apply`陷阱函数以保证每个参数都是数值。

出于安全的考虑，这段代码使用 `construct `陷阱抛出错误，以确保该函数不会被使用 new 运算符调用

> 实例对象 instance 对象会被同时判定为 proxy 与 target 对象的实例，是因为 instanceof 运算符使用了原型链来进行推断，而原型链查找并没有受到这个代理的影响，因此 proxy 对象与 target 对象对于 JS 引擎来说就有同一个原型。


## 4.3 调用类的构造函数

ES6 中新引入了`class`类的概念，类使用`constructor`构造函数封装数据，并规定必须始终使用 new 来调用，原因是类构造器的内部方法 [[Call]] 被明
确要求抛出错误。

代理可以拦截对于 [[Call]] 方法的调用，你可以借助代理调用的类构造器。例如在缺少 new 的情况下创建一个新实例，就使用 apply 陷阱函数实现：

```
class Person {
  constructor(name) {
    this.name = name;
  }
}
let PersonProxy = new Proxy(Person, {
  apply: function(trapTarget, thisArg, argumentList) {
    return new trapTarget(...argumentList);
  }
});
let me = PersonProxy("Jenny");
console.log(me.name); // "Jenny"
console.log(me instanceof Person); // true
console.log(me instanceof PersonProxy); // true
```

类构造器即类的构造函数，使用代理时它的行为就像函数一样，`apply`陷阱函数重写了默认的构造行为。

> 关于类的更多有趣的用法，可参考 [【ES6】更易于继承的类语法][es]

 [es]: https://segmentfault.com/a/1190000016897289

总结来说，代理的用途非常广泛，因为它提供了修改 JS 内置对象的所有行为的入口。上述例子只是简单的一些应用入门，还有更多复杂的示例，推荐阅读《深入理解ES6》。

#### 继续加油鸭少年！！！