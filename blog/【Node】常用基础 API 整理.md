# 一、Debug 调试方法

Node 的调试方法有很多，主要分为安装 node-inspect 包调试、用 Chrome DevTools 调试和 IDE 调试，可以在官网的 [Docs Debugging Guide](https://nodejs.org/en/docs/guides/debugging-getting-started/) 查看安装方法。


下面介绍使用 Chrome DevTools 调试的方法，首先安装 [Chrome Extension NIM](https://chrome.google.com/webstore/detail/nim-node-inspector-manage/gnhhdgbaldcilmgcpfddgdbkhjohddkj)，打开 Inspect 入口页面 [chrome://inspect](chrome://inspect) 

写一个简单 debug.js 测试文件：

```js
// apiTest/debug.js 
console.log("this is debug test")
function test () {
  console.log("hello world")
}
test()
```

使用`node --inspect-brk`来启动脚本，`-brk`相当于在程序入口前加一个断点，使得程序会在执行前停下来

```
$ node --inspect-brk apiTest/debug.js 
Debugger listening on ws://127.0.0.1:9229/44b5d11e-3261-4090-a18c-2d811486fd0a
For help, see: https://nodejs.org/en/docs/inspector
```
在 [chrome://inspect](chrome://inspect) 中设置监听端口 9229（默认），就可以看到可以 debug 的页面：

```js
(function (exports, require, module, __filename, __dirname) { 
console.log("this is debug test")
function test () {
  console.log("hello world")
}
test()
});
```

如果我们使用`node --inspect`来启动脚本，那整个代码直接运行到代码结尾，无法进行调试，但此时 Node 还进程没有结束，所以可以在 [http://127.0.0.1:9229/json/list](http://127.0.0.1:9229/json/list) 查询 devtoolsFrontendUrl ，复制此 Url 到 Chrome 上进行调试。

看到使用 Chrome DevTools 的调试方法还是比较复杂的，一些 IDE 都支持直接断点调试，推荐WebStorm、VScode。

# 二、全局变量

在 Node 中常用的全局方法有 CommonJS、Buffer、process、console、timer 等，这些方法不需要 `require`引入 API 就可以直接使用。

如果希望有属性或方法可以*“全局使用”*，那就将它挂载在 Node 的`global`对象上：

```js
global.gNum = 300
console.log(gNum); // 300
```

在 Node 中所有模块都可以使用这些全局变量，以下就介绍 Node 中的全局变量

## 2.1 CommonJS 模块

Node CommonJS 模块规范根据实现了`module`、`exports`和`require`模块机制。Node 对每个文件都被进行了模块封装，每个模块有自己的作用域，如在 debug 时看到的：

```js
(function (exports, require, module, __filename, __dirname) { 
	// some code
});
```

模块机制中的 `__dirname`、`__filename`、`exports`、`module`、`require()`这些变量虽然看起来是全局的，但其实它们仅存在于模块范围。需要注意的几点是：

- 模块内部`module`变量代表模块本身
- 模块提供`require()`方法引入外部模块到当前的上下文中
- `module.exports`属性代表模块对外接口，默认的快捷方式`exports`

简单的使用方式如下：

```js
/* common_exports.js */
exports.num = 100  
exports.obj = {
  a : 200
}
exports = {
  count : 300
}

/* common_require.js */
const mod = require('./common_exports')
console.log(mod) // { num: 100, obj: { a: 200 } }
console.log(mod.count)  // undefined
```

注意到上例中的`mod.count`为`undefined`，这是因为`exports`只是`module.exports`的引用，可以给`exports`添加属性，但不能修改`exports`的指向。

> 更深入的了解模块机制可以看 [【Node】前后端模块规范与模块加载原理](https://segmentfault.com/a/1190000018424385)

## 2.2 process 进程对象
process 包含了进程相关的属性和方法，Node 的 [process 文档](https://nodejs.org/api/process.html) 中的内容特别多，列举几个常用方法。

Node 进程启动时传递的参数都在 `process.arg `数组中：

```js
// process.js
const {argv , execPath} = process

argv.forEach((val, index) => {
  console.log(`${index}: ${val}`)
})
console.log(execPath)
```
可以在执行 process.js 时传递其他参数，这些参数都会保存在` argv `中：

```terminal
$ node apiTest/process.js one=1 --inspect --version
0: /usr/local/bin/node
1: /Users/mobike/Documents/webProjects/testNode/apiTest/process.js
2: one=1
3: --inspect
4: --version
/usr/local/bin/node
```
`process.argv`第一个参数就是 `process.execPath` ，即调用执行程序 Node 的路径，第二个参数时被执行的 JS 文件路径，剩下的就是自定义参数。

---

`process.env`是包含运行环境各种参数的对象，可以直接输出`env `查看所有参数信息，也可以输出某个属性：

```JS
const env = process.env
console.log(env.PATH) // /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/Users/Documents/webProjects/testNode/node_modules/.bin
console.log(env.SHELL) // /bin/zsh
```
在 webpack 打包过程中常用`process.env.NODE_ENV`判断生产环境或开发环境，`process.env`是没有`NODE_ENV`这个属性的，你可以在系统环境变量中配置，也可以在项目程序直接设置`process.env.NODE_ENV=‘dev’`。

---

`process.cwd()`方法返回 Node.js 进程的当前工作目录，和 Linus 命令`$ pwd`功能一样：

```js
// process.js
console.log(process.cwd()) // /Users/Documents/webProjects/testNode
```

```terminal
$ node process.js
/Users/WebstormProjects/testNode
$ pwd
/Users/WebstormProjects/testNode

```
## 2.3 Timers 异步

Node 中的计时器方法与 Web 浏览器中的JS 计时器类似，但内部实现是基于 Node 的 [Event Loop](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)。Node 中的计时器有`setImmediate()`、`setTimeout()`、`setInterval()`。

在 Node 中有一个轻量级的`process.nextTick()`异步方法，它是在当前事件队列结束时调用，
`setImmediate()`是当前 Node Event Loop 结束时立即执行，那执行顺序有什么区别呢？

下面举例说明`process.nextTick(fn)`与`setImmediate(fn)`与`setTimeout(fn,0)`之间的区别：

```js
// timer.js 
setImmediate(()=>{
  console.log("setImmediate")
});

setTimeout(()=>{
  console.log("setTimeout 0")
},0);

setTimeout(()=>{
  console.log("setTimeout 100")
},100);

process.nextTick(()=>{
  console.log("nextTick")
  process.nextTick(()=>{
    console.log("nextTick inner")
  })
});
```
看下执行结果: 

```
$ node timer.js 
nextTick
nextTick inner
setTimeout 0
setImmediate
setTimeout 100
```

`process.nextTick()`中的回调函数最快执行，因为它将异步事件插入到当前执行队列的末尾，但如果`process.nextTick()`中的事件执行时间过长，后面的异步事件就被延迟。

`setImmediate()`执行最慢，因为它将事件插入到下一个事件队列的队首，不会影响当前事件队列的执行。当`setTimeout(fn, 0)`是在`setImmediate()`之前执行。

## 2.4 Buffer 二进制

Buffer 对象用于处理二进制数据流。JS 没有处理二进制的功能，而 Node 中的一部分代码是由 C++ 实现的，所有 Node 中的 Buffer 性能部分用 C++ 实现，非性能部分由 JS 封装。

Buffer 实例类似整数数组，元素为十六进制的两位数（0～255），并且挂载在 global 对象上不需要 require就能使用。

最新的 Buffer API 使用`Buffer.alloc(length, value)`创建固定长度为 length 的 Buffer 实例，value 默认填充 0，使用`Buffer.from()`将其它类型数据转为 Buffer：

```js
console.log(Buffer.alloc(5))        // <Buffer 00 00 00 00 00>
console.log(Buffer.alloc(5, 44))    // <Buffer 2c 2c 2c 2c 2c>
console.log(Buffer.from([3, 4, 5])) // <Buffer 03 04 05>
console.log(Buffer.from('test'))    // <Buffer 74 65 73 74>
console.log(Buffer.from('测试'))     // <Buffer e6 b5 8b e8 af 95>
```

注意到字符串转 Buffer 时英文占一位，中文占三位，而不是四位，当中文乱码的时可以考虑没有正确读取 Buffer 流。

Buffer 类提供几个静态方法，`Buffer.byteLength()`计算长度，`Buffer.isBuffer()`做验证，`Buffer.concat()`拼接 Buffer 实例:

```
const buf1 = Buffer.from([3, 4, 5])
const buf2 = Buffer.from('test')

console.log(Buffer.byteLength('test'))   // 4
console.log(Buffer.byteLength('测试'))    // 6 
console.log(Buffer.isBuffer('test'))     // false
console.log(Buffer.isBuffer(buf1))       // true
console.log(Buffer.concat([buf1, buf2])) // <Buffer 03 04 05 74 65 73 74>

```

除此之外，Buffer 实例也有常用的属性和方法，类似 JS 中的 String，有`length`、`toString('base64')`、`equals()`、`indexOf()`等。


> 以上就是 Node 全局变量的概述，其他的 API 或内置模块都需要·
`require('xxx')`引入使用，我们可以在 [nodejs.cn](http://nodejs.cn/) 中查看关于 [Global API](http://nodejs.cn/api/globals.html) 更详细的介绍。

# 三、基础 API



## 3.1 path 路径相关

path 是处理和路径相关问题的内置 API，可以直接`require('path')`使用。以下示例常用的 path 方法。

对路径的处理常用`path.normalize()`规范路径、`path.join()`拼接路径，以及使用`path.resolve()`将相对路径解析为绝对路径：

```js
const path = require('path')
console.log(
  path.normalize('//asd\/das'), // /asd/das
  path.join('user', 'local'),   // user/local
  path.resolve('./'))           // /Users/Documents/webProjects/testNode/apiTest
```

解析某个路径，可以用`path.basename()`得到文件名称，`path.extname()`得到后缀扩展名，`path.dirname()`得到目录名：

```js
const path = require('path')
const filePath = 'webProjects/testNode/apiTest/path.js'
console.log(
  path.basename(filePath), // path.js 
  path.extname(filePath)   // .js 
  path.dirname(filePath),  // webProjects/testNode/apiTest
)
```

以上解析路径方法得到某个值，还可以使用`path.parse()`完全解析路径为一个对象，`path.format()`反向操作:

```js
let sp = path.parse(filePath)
console.log(sp)
// { root: '',
//   dir: 'webProjects/testNode/apiTest',
//   base: 'path.js',
//   ext: '.js',
//   name: 'path' }
console.log(path.format(sp))
// webProjects/testNode/apiTest/path.js
```

 除此之外，还有对于系统路径的操作，使用`path.sep`取得路径分隔符，路径片段分隔符，POSIX 上是` /`, Windows 上是` \`，`path.delimiter `取得系统路径定界符，POSIX 上是`: `，Windows 上是`;`	，示例如下：
 
```js
console.log(filePath.split(path.sep)) 
// [ 'webProjects', 'testNode', 'apiTest', 'path.js' ]

console.log(process.env.PATH) // 系统路径配置
// /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin

console.log(process.env.PATH.split(path.delimiter))
// [ '/usr/local/bin', '/usr/bin', '/bin', '/usr/sbin', '/sbin' ]
```

以上就是 Node 对路径的常用操作，需要注意的是，在获取路径时有几种方式，得到的路径是不同的：


- `__dirname` 、`__filename `：总是返回文件绝对路径；
- `process.cwd()` 或 `$ pwd` ：返回执行 Node 命令的文件夹；
- `path.resolve('./')`：是相对 Node 启动文件夹，在`require()`中`./`是相对于当前文件夹；
 

## 3.3 events 事件

大部分 Node API 都采用异步事件驱动，所有能触发事件对象都是 EventEmitter 类的实例，通过 `EventEmitter.on() `绑定事件，然后通过 `EventEmitter.emit()` 触发事件。

```js
// apiTest/events.js
const Events = require('events')

class MyEvents extends Events{
}
const event = new MyEvents()

event.on('test-event',()=>{
  console.log('this is an event')
})

event.emit('test-event')
setInterval(()=>{
  event.emit('test-event')
},500)
```
执行以上代码会一直连续处罚 *test-event* 事件，当然还可以**传递事件参数**，并且可以传递多个参数。修改上诉代码如下：

```
event.on('test-event', (data, time) => {
  console.log(data,time)
})
event.emit('test-event', [1, 2, 3], new Date())
```

```
$ node apiTest/events.js
[ 1, 2, 3 ] 2019-04-23T07:28:00.420Z
```

同一个事件监听器可以绑定多个事件，触发时按照绑定顺序加入执行队列，并且可以使用`EventEmitter.removeListener()`删除监听器的事件：

```
function fn1 () {
  console.log('fn1')
}
function fn2 () {
  console.log('fn2')
}
event.on('multi-event',fn1)
event.on('multi-event',fn2)

setInterval(()=>{
  event.emit('multi-event')
},500)
setTimeout(()=>{
  event.removeListener('multi-event', fn2)
}, 600)
```
```
$ node apiTest/events.js
[ 1, 2, 3 ] 2019-04-23T07:39:11.624Z
fn1
fn2
fn1
fn1
...
```

## 3.4 fs 文件系统

Node 文件模块通过` require('fs) `使用，所用方法都有同步和异步方法。

文件系统中的异步方法，第一个参数保留给异常，操作成功时参数值为`null`或`undefined`，最后一个参数就是回调函数。例如读取文件的`fs.readFile()`和写文件的`fs.writeFile()`示例如下：
 
 ```
const fs = require('fs')

fs.readFile('./apiTest/fs.js', (err, data) => {
  if (err) throw err
  console.log('readFile done!!!')
})

fs.writeFile('./apiTest/fs.txt', 'this is test file', {
  encoding: 'utf8'
}, (err) => {
  if (err) throw err
  console.log('writeFile done!!!')
})
 ```

> 推荐 [nodejs.cn](http://nodejs.cn/) 中的 [Docs API 中文版](http://nodejs.cn/api/)查看更多 Node API 的使用。