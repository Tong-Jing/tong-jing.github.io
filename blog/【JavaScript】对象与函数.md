# `prototype `与 `__proto__`

```
function foo () {}

let obj = {}

```
任意 obj 都有 `__proto__`属性，

每次创建新函数时默认为函数添加`prototype`属性，这个属性是一个指针，指向包含所有实例共享的属性和方法的对象，

在创建函数后，原型对象回取得 constructor 属性值指向构造函数本身，而其他方法都是通过 Object 继承而来，所以每个实例都有`[[prototype]]`内部指针,外部通过`__proto__`属性访问，这个指针指向构造函数的原型对象。


```
console.log(Object.getPrototypeOf(pfoo) == pfoo.__proto__) // true
```

`foo.__proto__` 是 Function，属于 [native code]

`obj.__proto__`是 Object 类型

# Function
Function 本事是一个语法糖，想当于 new

### new 操作符
只要通过new调用的函数，都可以作为构造函数
- 创建一个对象
- 链接到原型（将构造函数的作用域this赋给新对象）
- 为新对象添加属性值
- 返回新对象

