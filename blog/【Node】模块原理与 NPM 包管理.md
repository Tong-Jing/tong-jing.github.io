> CommonJS 定义了 module、exports 和 require 模块规范，Node.js 为了实现这个简单的标准，从底层 C/C++ 到 JavaScript，从路径分析、文件定位到编译执行，经历了一系列复杂的过程。简单的了解 Node 模块的原理，有利于我们重新认识基于 Node 搭建的框架。

# 一、CommonJS 模块规范
CommonJS 规范或标准简单来说是一种理论，它期望 JavaScript 可以具备跨宿主环境执行的能力，不仅可以开发客户端应用，还可以开发服务端应用、命令行工具、桌面图形界面应用等。


CommonJS 规范对模块的定义分为三个部分：

- **模块定义**

  在模块中存在 module 对象代表模块本身，模块上下文提供 exports 属性 ，将方法挂载在 exports 对象上即可以定义导出方式，例如：
  
  ```JavaScript
  	// math.js
 	exports.add = function(){ //...}
  ```

- **模块引用**

   module 提供 require() 方法引入外部模块的API到当前的上下文中：
   
  ```JavaScript
 	var math = require('math')
  ```

- **模块标识**

  模块标识实际就是传递给`require()`方法中的参数，可以是按小驼峰（camelCase）命名的字符串，也可以是文件路径。
  

Node.js 借鉴了 CommonJS 规范的设计，特别是 CommonJS 的 **Modules 规范**，实现了一套模块系统，同时 NPM 实现了 CommonJS 的 **Packages 规范**，模块和包组成了 Node 应用开发的基础。



# 二、Node 模块加载原理

上述模块规范看起来十分简单，只有 module、exports 和 require，但 Node 是如何实现的呢？

需要经历路径分析、文件定位、编译执行三个步骤。

## 2.1 路径分析

回顾`require()`接收 **模块标识** 作为参数来引入模块，Node 就是基于这个标识符进行路径分析。不同的标识符采用的分析方式是不同的，主要分为一下几类：

- ### Node提供的**核心模块**，如http、fs、path
  
  核心模块在Node源码编译时存为二进制执行文件，在 Node 启动时直接加载到内存中，路径分析中优先判断，所以加载速度很快，而且也不用后续的文件定位和编译执行。
  
  > 如果想加载与核心模块同名的自定义模块，如自定义http模块，那必须选用不同标志符或改用路径方式

- ### 路径形式的**文件模块**，`.、..`相对路径模块和`/`绝对路径模块


  以`.、..或/`开始的标识符都会当成文件模块处理，Node 会将`require()`中的路径转为真实路径作为索引，然后编译执行。
  
  由于文件模块明确了文件位置，所以缩短了路径分析时间，加载速度仅慢与核心模块。

- ### **自定义模块**，即非路径形式的文件模块
  
  即不是核心模块，也不是路径形式的文件模块，自定义文件是特殊的文件模块，在路径查找时Node会逐级查找该模块路径中的路径，**模块路径查找策略**示例如下：
  
  ```
  // paths.js
  console.log(module.paths)
  
  // Terminal
  $ node paths.js
  [ '/Users/tong/WebstormProjects/testNode/node_modules',
  '/Users/tong/WebstormProjects/node_modules',
  '/Users/tong/node_modules',
  '/Users/node_modules',
  '/node_modules' ]

  ```
 
 从上述示例输出的模块路径数组可以看出，模块的查找时沿当前路径向上逐级查找 node_modules 目录，直到目标路径为止，类似 JS 原型链或作用域链。路径越深速度越慢，所以自定义模块加载速度最慢。


> **缓存优先机制**：Node 会对引入过的模块进行缓存以提高性能，不同于浏览器缓存的是文件，Node 缓存的是编译和执行后的对象，所以`require()`对相同模块的二次加载采用缓存优先的方式。这个缓存优先是第一优先级的，比核心模块的优先级要高！

## 2.2 文件定位

模块路径分析完成后是文件定位，主要包括文件扩展名的分析、目录和包的处理。为了表达的更清晰，将文件定位分为四个步骤：
	
- #### step1: 补充扩展名

	通常`require()`中的标识符是不包含文件扩展名的，这种情况下，**Node会按照.js、.json、.node 的顺序尝试补充扩展名**。

	> 在尝试补充扩展名时，需要调用fs模块同步阻塞式判断文件是否存在，所以这里提升性能的小技巧，就是.json和.node文件传递给`require()`时带上扩展名会加快一些速度。

- #### step2: 目录处理查找 pakage.json

	如果补充扩展名后没有找到对应文件，但是得到了一个目录，此时**Node会将目录当做一个包处理**。依据CommonJS包规范的实现，Node会在目录下查找`pakage.json`（包描述文件），通过`JSON.parse()`解析成包描述对象，从中**取`main`属性指定的文件名定位**。
	
- #### step3: 继续默认查找 index 文件

	如果没有`pakage.json`或者`main`属性指定的文件名错误，那 Node 会将index当做默认文件名，依次查找 index.js、index.json、index.node

- #### step4: 进入下一个模块路径

	在上述目录分析过程中没有成功定位时，自定义模块按路径查找策略进入上一层 node_modules 目录，当整个模块路径数组遍历完毕后没有定位到文件，则会抛出查找失败异常。

> 缓存加载的优化策略使得二次引入不需要路径分析、文件定位、编译执行这些过程，而且核心模块也不需要文件定位的过程，这大大提高了再次加载模块时的效率

## 2.3 编译执行

Node 中每个模块都是一个对象，在具体定位到文件后，Node 会新建该模块对象，然后根据路径载入并编译。**不同的文件扩展名载入方法**为：

- **.js 文件**: 通过 fs 模块同步读取后编译执行
- **.json 文件**: 通过 fs 模块同步读取后，用`JSON.parse()`解析并返回结果
- **.node 文件**: 这是用 C/C++ 写的扩展文件，通过`process.dlopen()`方法加载最后编译生成的
- **其他扩展名**: 都被当做 js 文件载入

载入成功后 Node 会调用具体的编译方式将文件执行后返回给调用者。对于 .json 文件的编译最简单，`JSON.parse()`解析得到对象后直接赋值给模块对象的`exports`，而 .node 文件是C/C++编译生成的，Node 直接调用`process.dlopen()`载入执行就可以，下面重点介绍 .js 文件的编译。

---

在 CommonJS 模块规范中有`module`、`exports` 和 `require` 这3个变量，在 Node API 文档中每个模块还有 `__filename`、`__dirname`这两个变量，但是在模块中没有定义这些变量，那它们是怎么产生的呢？

事实上在编译过程中，Node 对每个 JS 文件都被进行了封装，例如一个 JS 文件会被封装成如下：

```
(function (exports, require, module, __filename, __dirname) {
	var math = require('math')
	export.add = function(){ //... }
})
```

首先每个模块文件之间都进行了**作用域隔离**，通过vm原生模块的`runInThisContext()`方法（类似 eval）返回一个具体的 function 对象，最后将当前模块对象的`exports`属性、`require()`方法、模块对象本身`module`、文件定位时得到的**完整路径**`__filename`和**文件目录**`__dirname`作为参数传递给这个 function 执行。模块的`exports`属性上的任何方法和属性都可以被外部调用，其余的则不可被调用。

至此，`module`、`exports` 和 `require`的流程就介绍完了。

---

曾经困惑过，每个模块都可以使用`exports`的情况下，为什么还必须用`module.exports`。

这是因为`exports`在编译过程中时通过形参传入的，直接给`exports`形参赋值只改变形参的引用，不能改变作用域外的值，例如：

```
let change = function (exports) {
  exports = 100
  console.log(exports)
}

var exports = 2
change(exports) // 100
console.log(exports) // 2
```
所以直接赋值给`module.exports`对象就不会改变形参的引用了。

> 编译成功的模块会将文件路径作为索引缓存在 `Module._cache` 对象上，路径分析时优先查找缓存，提高二次引入的性能。


# 三、Node 核心模块

总结来说 Node 模块分为Node提供的**核心模块**和用户编写的**文件模块**。文件模块是在运行时动态加载，包括了上述完整的路径分析、文件定位、编译执行这些过程，核心模块在Node源码编译成可执行文件时存为二进制文件，直接加载在内存中，所以不用文件定位和编译执行。

**核心模块分为 C/C++ 编写的和 JavaScript 编写的两部分**，在编译所有 C/C++ 文件之前，编译程序需要将所有的 JavaScript 核心模块编译为 C/C++ 可执行代码，编译成功的则放在 `NativeModule._cache`对象上，显然和文件模块 `Module._cache`的缓存位置不同。

在核心模块中，有些模块由纯 C/C++ 编写的**内建模块**，主要提供 API 给 JavaScript 核心模块，通常不能被用户直接调用，而有些模块由  C/C++ 完成核心部分，而 JavaScript 实现封装和向外导出，如 buffer、fs、os 等。

所以在Node的模块类型中存在**依赖层级关系**：内建模块（C/C++）—> 核心模块（JavaScript）—> 文件模块。

使用`require()`十分的方便，但从 JavaScript 到 C/C++ 的过程十分复杂，总结来说需要经历 C/C++ 层面内建模块的定义、（JavaScript）核心模块的定义和引入以及（JavaScript）文件模块的引入。


# 四、其它模块规范

对比前后端的 JavaScript，浏览器端的 JavaScript 需要经历从同一个服务器端分发到多个客户端执行，通过网络加载代码，瓶颈在于宽带；而服务器端 JavaScript 相同代码需要多次执行，通过磁盘加载，瓶颈在于 CPU 和内存，所以前后端的 JavaScript 在 Http 两端的职责完全不用。

Node 模块的引入几乎是同步的，而前端模块如果同步引入，那脚本加载需要太长的时间，所以 CommonJS 为后端 JavaScript 制定的规范不适合前端。而后出现 AMD 和 CMD 用于前端应用场景。

## AMD 规范
 AMD 即异步模块定义（Asynchronous Module Definition）,模块定义为：
 
 ```
 define(id?, dependencies?, factory);
 ```
 AMD 模块需要用 define 明确定义一个模块，其中模块 id 与依赖 dependencies 是可选的，factory的内容就是实际代码的内容。例如指定一些依赖到模块中：
 
 ```
 define(['dep1', 'dep2'], function(){
 	// module code
 });
 ```
 
require.js 实现 AMD 规范的模块化，感兴趣的可以查看 require.js 的文档。

## CMD 规范

CMD 模块的定义更加简单：

```
 define(factory);
```

定义的模块同 Node 模块一样是隐式包装，在依赖部分支持动态引入，例如：

```
 define(function(require, exports, module){
 	// module code
 });
```
require、exports、module 通过形参传递给模块，需要依赖模块时直接使用 require() 引入。

sea.js 实现 AMD 规范的模块化，感兴趣的可以查看 sea.js 的文档。


-
##### *推荐两本 Node 的书籍：《Node.js 实战》主要是使用示例，《深入浅出 Node.js》偏实现原理。*

###### 当然我的博客也会继续总结更新，下一篇内容会是关于 CommonJS 包规范和 NPM 包管理的内容。
-


## ES6 模块语法



# 四、CommonJS 包规范

# 五、NPM 包管理

使用基于 Node 的程序时都注意到，Node 依赖项在程序的根目录下的 package.json文件中，它包含一些JSON表达式，并遵循CommonJS包描述标准
package.json文件用于描述你的应用程序，

## 安装依赖包
使用`npm install`会出现node_modules 目录

## NPM 实现过程，木易杨总结的




