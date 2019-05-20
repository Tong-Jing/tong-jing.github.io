

# 零、准备工作

##  github地址

## 1.1 .gitignore

[gitignore](https://git-scm.com/docs/gitignore) 是 git 忽略提交规则。

## 1.2 editorConfig

[EditorConfig](https://editorconfig.org/) 是团队协作必须的代码规范配置。

## 1.3 ESLint

[ESLint](https://cn.eslint.org/) 是 JS 代码检查工具

```
$ npm i eslint
$ eslint --init
```

```
// package.json
"scripts": {
    "lint": "eslint ."
  }
```
pre-commit 在git提交前执行 scripts 中的命令，这个插件只安装在开发环境中即可：

```
$ npm install pre-commit -D
```


# 一、创建 HTTP Server 服务器


Node 的 http 模块提供 HTTP 服务器和客户端接口，通过` require('http')`使用。

我们来创建一个简单的 http server。配置参数如下：

```js
// server/config.js
module.exports = {
  root: process.cwd(),
  host: '127.0.0.1',
  port: '8877'
}
```
process.cwd()方法返回 Node.js 进程的当前工作目录，和 Linus 命令`pwd`功能一样，


Node 服务器每次收到 HTTP 请求后都会调用 http.createServer() 这个回调函数，每次收一条请求，都会先解析请求头作为新的 request 的一部分，然后用新的 request 和 respond 对象触发回调函数。以下创建一个简单的 http 服务，先默认响应的 status 为 200：

```js
// server/http.js
const http = require('http')
const path = require('path')

const config = require('./config')

const server = http.createServer((request, response) => {
  let filePath = path.join(config.root, request.url)
  response.statusCode = 200
  response.setHeader('content-type', 'text/html')
  response.write(`<html><body><h1>Hello World! </h1><p>${filePath}</p></body></html>`)
  response.end()
})

server.listen(config.port, config.host, () => {
  const addr = `http://${config.host}:${config.port}`
  console.info(`server started at ${addr}`)
})
```

客户端请求静态资源的地址可以通过`request.url`获得，然后使用 path 模块拼接资源的路径。

执行`$ node server/http.js` 后访问 http://127.0.0.1:8877/ 后的任意地址都会显示该路径：
`





每次修改服务器响应内容，都需要重新启动服务器更新，推荐自动监视更新自动重启的插件supervisor，使用supervisor启动服务器。

```
$ npm install supervisor -D
$ supervisor server/http.js

```



#二、使用 fs 读取资源文件

我们的目的是搭建一个静态资源服务器，当访问一个到资源文件或目录时，我们希望可以得到它。这时就需要使用 Node 内置的 fs 模块读取静态资源文件，

，使用 `fs.stat() `读取文件状态信息，通过回调中的状态`stats.isFile()`判断文件还是目录，并使用`fs.readdir()`读取目录中的文件名

```
// server/route.js
const fs = require('fs')

module.exports = function (request, response, filePath){
  fs.stat(filePath, (err, stats) => {
    if (err) {
      response.statusCode = 404
      response.setHeader('content-type', 'text/plain')
      response.end(`${filePath} is not a file`)
      return;
    }
    if (stats.isFile()) {
      response.statusCode = 200
      response.setHeader('content-type', 'text/plain')
      fs.createReadStream(filePath).pipe(response)
    } 
    else if (stats.isDirectory()) {
      fs.readdir(filePath, (err, files) => {
        response.statusCode = 200
        response.setHeader('content-type', 'text/plain')
        response.end(files.join(','))
      })
    }
  })
}
```
其中`fs.createReadStream()`读取文件流，`pipe()`是分段读取文件到内存，优化高并发的情况。

修改之前的 http server ，引入上面新建的 route.js 作为响应函数：

```
// server/http.js
const http = require('http')
const path = require('path')

const config = require('./config')
const route = require('./route')

const server = http.createServer((request, response) => {
  let filePath = path.join(config.root, request.url)
  route(request, response, filePath)
})

server.listen(config.port, config.host, () => {
  const addr = `http://${config.host}:${config.port}`
  console.info(`server started at ${addr}`)
})
```

再次执行 `$ node server/http.js` 

> 成熟的静态资源服务器 anywhere，深入理解 nodejs 作者写的。

#三、util.promisify 优化 fs 异步

我们注意到`fs.stat()`和`fs.readdir()` 都有 callback 回调。我们结合 Node 的 util.promisify() 来链式操作，代替地狱回调。

util.promisify 只是返回一个 Promise 实例来方便异步操作，并且可以和 async/await 配合使用，修改 route.js 中 fs 操作相关的代码：

```
// server/route.js
const fs = require('fs')
const util = require('util')

const stat = util.promisify(fs.stat)
const readdir = util.promisify(fs.readdir)

module.exports = async function (request, response, filePath) {
  try {
    const stats = await stat(filePath)
    if (stats.isFile()) {
      response.statusCode = 200
      response.setHeader('content-type', 'text/plain')
      fs.createReadStream(filePath).pipe(response)
    }
    else if (stats.isDirectory()) {
      const files = await readdir(filePath)
      response.statusCode = 200
      response.setHeader('content-type', 'text/plain')
      response.end(files.join(','))
    }
  } catch (err) {
    console.error(err)
    response.statusCode = 404
    response.setHeader('content-type', 'text/plain')
    response.end(`${filePath} is not a file`)
  }
}

```
因为 `fs.stat()`和`fs.readdir()` 都可能返回 error，所以使用 try-catch 捕获。

使用异步时需注意，异步回调需要使用 await 返回异步操作，不加 await 返回的是一个 promise，而且 await 必须在async里面使用。



#四、添加模版引擎

从上面的例子是手工输入文件路径，然后返回资源文件。现在优化这个例子，将文件目录变成 html 的 a 链接，点击后返回文件资源。

在第一个例子中使用`response.write()`插入 HTML 标签，这种方式显然是不友好的。这时候就使用模版引擎做到拼接 HTML。

常用的模版引擎有很多，ejs、jade、handlebars，这里的使用ejs：

```
npm i ejs
```

新建一个模版 src/template/index.ejs	，和 html 文件很像：

```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Node Server</title>
</head>
<body>
<% files.forEach(function(name){ %>
  <a href="../<%= dir %>/<%= name %>"> <%= name %></a><br>
<% }) %>
</body>
</html>
```

再次修改 route.js，添加 ejs 模版并`ejs.render()`，在文件目录的代码中传递 files、dir 等参数：

```
// server/route.js

const fs = require('fs')
const util = require('util')
const path = require('path')
const ejs = require('ejs')
const config = require('./config')
// 异步优化
const stat = util.promisify(fs.stat)
const readdir = util.promisify(fs.readdir)
// 引入模版
const tplPath = path.join(__dirname,'../src/template/index.ejs')
const sourse = fs.readFileSync(tplPath) // 读出来的是buffer

module.exports = async function (request, response, filePath) {
  try {
    const stats = await stat(filePath)
    if (stats.isFile()) {
      response.statusCode = 200
      ···
    }
    else if (stats.isDirectory()) {
      const files = await readdir(filePath)
      response.statusCode = 200
      response.setHeader('content-type', 'text/html')
      // response.end(files.join(','))

      const dir = path.relative(config.root, filePath) // 相对于根目录
      const data = {
        files,
        dir: dir ? `${dir}` : '' // path.relative可能返回空字符串（）
      }

      const template = ejs.render(sourse.toString(),data)
      response.end(template)
    }
  } catch (err) {
    response.statusCode = 404
    ···
  }
}
```

重启动`$ node server/http.js` 就可以看到文件目录的链接：



#五、匹配文件 MIME 类型

静态资源有图片、css、js、json、html等，
在上面判断`stats.isFile()`后响应头设置的 Content-Type 都为 text/plain，但各种文件有不同的 Mime 类型列表。

我们先根据文件的后缀匹配它的 MIME 类型：

```
// server/mime.js
const path = require('path')
const mimeTypes = {
  'js': 'application/x-javascript',
  'html': 'text/html',
  'css': 'text/css',
  'txt': "text/plain"
}

module.exports = (filePath) => {
  let ext = path.extname(filePath)
    .split('.').pop().toLowerCase() // 取扩展名

  if (!ext) { // 如果没有扩展名，例如是文件
    ext = filePath
  }
  return mimeTypes[ext] || mimeTypes['txt']
}
```

匹配到文件的 MIME 类型，再使用`response.setHeader('Content-Type', 'XXX')`设置响应头：


```
// server/route.js
const mime = require('./mime')
···
    if (stats.isFile()) {
      const mimeType = mime(filePath)
      response.statusCode = 200
      response.setHeader('Content-Type', mimeType)
      fs.createReadStream(filePath).pipe(response)
    }
```

运行 server 服务器访问 JS 文件，可以看到 Content-Type 修改了




# 六、文件传输压缩

注意到 request header 中有 Accept—Encoding：gzip，deflate，告诉服务器客户端所支持的压缩方式，响应时 response header 中使用 content-Encoding 标志文件的压缩方式。


node 内置 zlib 模块支持文件压缩。在前面文件读取使用的是`fs.createReadStream()`，所以压缩是对 ReadStream 文件流。示例 gzip，deflate 方式的压缩：


最常用文件压缩，gzip等，使用，对于文件是用ReadStream文件流进行读取的，所以对ReadStream进行压缩：


```js
// server/compress.js
const  zlib = require('zlib')

module.exports = (readStream, request, response) => {
  const acceptEncoding = request.headers['accept-encoding']
  
  if (!acceptEncoding || !acceptEncoding.match(/\b(gzip|deflate)\b/)) {
    return readStream
  }
  else if (acceptEncoding.match(/\bgzip\b/)) {
    response.setHeader("Content-Encoding", 'gzip')
    return readStream.pipe(zlib.createGzip())
  }
  else if (acceptEncoding.match(/\bdeflate\b/)) {
    response.setHeader("Content-Encoding", 'deflate')
    return readStream.pipe(zlib.createDeflate())
  }
}
```

修改 route.js 文件读取的代码：

```
// server/route.js
const compress = require('./compress')
···
  if (stats.isFile()) {
      const mimeType = mime(filePath)
      response.statusCode = 200
      response.setHeader('Content-Type', mimeType)
      
      // fs.createReadStream(filePath).pipe(response)
+     let readStream = fs.createReadStream(filePath)
+     if(filePath.match(config.compress)) { // 正则匹配：/\.(html|js|css|md)/
        readStream = compress(readStream,request, response)
      }
      readStream.pipe(response)
    }
```
运行 server 可以看到不仅 response header 增加压缩标志，而且 3K 大小的资源压缩到了 1K，效果明显：


# 七、资源缓存

以上的 Node 服务都是浏览器首次请求或无缓存状态下的，那如果浏览器/客户端请求过资源，一个重要的前端优化点就是缓存资源在客户端。**缓存有强缓存和协商缓存**：

**强缓存**在 Request Header 中的字段是 Expires 和 Cache-Control；如果在有效期内则直接加载缓存资源，状态码直接是显示 200。

**协商缓存**在 Request Header 中的字段是：

- If-Modified-Since（对应值为上次 Respond Header 中的 Last-Modified）
- If-None—Match（对应值为上次 Respond Header 中的 Etag）

如果协商成功则返回 304 状态码，更新过期时间并加载浏览器本地资源，否则返回服务器端资源文件。

首先配置默认的 cache 字段：

```
// server/config.js
module.exports = {
  root: process.cwd(),
  host: '127.0.0.1',
  port: '8877',
  compress: /\.(html|js|css|md)/,
  cache: {
    maxAge: 2,
    expires: true,
    cacheControl: true,
    lastModified: true,
    etag: true
  }
}
```

新建 server/cache.js，设置响应头：

```
const config = require('./config')
function refreshRes (stats, response) {
  const {maxAge, expires, cacheControl, lastModified, etag} = config.cache;

  if (expires) {
    response.setHeader('Expires', (new Date(Date.now() + maxAge * 1000)).toUTCString());
  }
  if (cacheControl) {
    response.setHeader('Cache-Control', `public, max-age=${maxAge}`);
  }
  if (lastModified) {
    response.setHeader('Last-Modified', stats.mtime.toUTCString());
  }
  if (etag) {
    response.setHeader('ETag', `${stats.size}-${stats.mtime.toUTCString()}`); // mtime 需要转成字符串，否则在 windows 环境下会报错
  }
}

module.exports = function isFresh (stats, request, response) {
  refreshRes(stats, response);

  const lastModified = request.headers['if-modified-since'];
  const etag = request.headers['if-none-match'];

  if (!lastModified && !etag) {
    return false;
  }
  if (lastModified && lastModified !== response.getHeader('Last-Modified')) {
    return false;
  }
  if (etag && etag !== response.getHeader('ETag')) {
    return false;
  }
  return true;
};
```
 最后修改 route.js 中的
 
```
// server/route.js
+ const isCache = require('./cache')

   if (stats.isFile()) {
      const mimeType = mime(filePath)
      response.setHeader('Content-Type', mimeType)

+     if (isCache(stats, request, response)) {
        response.statusCode = 304;
        response.end();
        return;
      }
      
      response.statusCode = 200
      // fs.createReadStream(filePath).pipe(response)
      let readStream = fs.createReadStream(filePath)
      if(filePath.match(config.compress)) {
        readStream = compress(readStream,request, response)
      }
      readStream.pipe(response)
    }
``` 	

重启 node server 访问某个文件，在第一次请求成功时 Respond Header 返回缓存时间：

一段时间后再次请求该资源文件，Request Header 发送协商请求字段：


以上就是一个简单的 Node 静态资源服务器。