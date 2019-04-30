

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


# 一、HTTP 服务

## 1.1 createServer()

Node 的 http 模块提供 HTTP 服务器和客户端接口，通过 require('http')使用

Node 服务器每次收到 HTTP 请求后都会调用 http.createServer() 这个回调函数

Node 服务器每次收一条请求，都会先解析请求头作为新的 request 的一部分，然后用新的 request 和 respond 对象触发回调函数。

```js
// ../src/helper/route.js
module.exports = {
  root: process.cwd(),
  host: '127.0.0.1',
  port: '8877'
}
```

```js

// ../src/app.js
const http = require('http')
const config = require('../config/httpConfig.js')

const server = http.createServer((request, response) => {
  response.statusCode = 200
  response.setHeader('content-type', 'text/html')
  response.write('<html><body><h1>Hello World! </h1></body></html>')
  response.end()

})
server.listen(config.port, config.host, () => {
  const addr = `http://${config.host}:${config.port}`
  console.info(`server started at ${addr}`)
})
```

每次修改服务器响应内容，都需要重新启动服务器更新，推荐自动监视更新自动重启的插件supervisor，使用supervisor启动服务器。

```
$ npm install supervisor -D
$ supervisor server/http.js

```

客户端请求静态资源的地址可以通过`request.url`获得，然后使用 path 模块拼接资源的路径

#二、fs 输出资源


修改 http.createServer()，使用 fs 模块读取静态资源文件，

fs.stat 读取文件信息

```
// ../src/app.js

const fs = require('fs')

const server = http.createServer((request, response) => {
  const filePath = path.join(config.root, request.url)
  
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
    } else if (stats.isDirectory()) {
      fs.readdir(filePath, (err, files) => {
        response.statusCode = 200
        response.setHeader('content-type', 'text/plain')
        response.end(files.join(','))
      })
    }
  })
})
```
fs.createReadStream 读取文件流，pipe是分段读取文件到内存，优化高并发的情况，

> 成熟的静态资源服务器anywhere，深入理解nodejs作者写的

#三、util.promisify 异步优化



```
// ../src/helper/route.js
const fs = require('fs')
const util = require('util')

const stat = util.promisify(fs.stat)
const readdir = util.promisify(fs.readdir)

module.exports = async function (request, response, filePath) {
  try {
    const stats = await stat(filePath) // await 必须在async 里面
    if (stats.isFile()) {
      response.statusCode = 200
      // ...
    }
    else if (stats.isDirectory()) {
      const files = await readdir(filePath)
      response.statusCode = 200
      // ...
    }
  } catch (err) {
    console.error(err)
    response.statusCode = 404
    // ...
  }
}

```
使用异步时需注意，异步回调需要使用await返回异步操作，不加await返回的是一个promise，而且await必须在async里面使用
```
// server/http.js
const file = require('../config/file')

const server = http.createServer((request, response) => {
  let filePath = path.join(httpConfig.root, request.url)
  file(request, response, filePath)
})
```



#四、模版引擎
从上面的例子是手工输入文件路径，然后返回资源文件，

优化一下，文件夹变成html的a链接，点击后返回文件资源。这时候就使用模版引擎做到拼接html。示例 handlebars 模版引擎


```
// ../src/helper/route.js

const tplPath = path.join(__dirname,'../template/dir.tpl')
const sourse = fs.readFileSync(tplPath) // 读出来的是buffer
const template = Handlebars.compile(sourse.toString()) // compile 参数是字符串

else if (stats.isDirectory()) {
      const files = await readdir(filePath)
      response.statusCode = 200
      response.setHeader('content-type', 'text/html')

      const dir = path.relative(config.root, filePath) // 相对于根目录

      const data = {
        title: path.basename(filePath),
        files: files,
        dir: dir ? `/${dir}` : ''// /表示从跟路径开始，path.relative可能返回空字符串（）
      }

      // response.end(files.join(','))
      response.end(template(data))
    }
```
template/dir.tpl

```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>{{title}}</title>
</head>
<body>
{{#each files}}
<a href="{{../dir}}/{{this}}">{{this}}</a>
{{/each}}
</body>
</html>

```

#五、资源类型 Content-Type

设置 Content-Type

response.setHeader('Content-Type', 'text/html') 设置响应头，但对于各种文件有不同的 Mime 类型列表

```
// helper/mime.js
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

```
// helper/mime.js
	if (stats.isFile()) {
      const contentType = mimeType(filePath) // 查询文件 content type
      response.statusCode = 200
      response.setHeader('Content-Type', contentType)
      fs.createReadStream(filePath).pipe(response)
    }

```

还可以根据文件类型设置图片

```
const data = {
        title: path.basename(filePath),
        files: files.map((file)=>{
          return {
            file,
            icon: mimeType(file)
          }
        }),
        dir: dir ? `/${dir}` : ''// /表示从跟路径开始，path.relative可能返回空字符串（）
      }
```

在模版引擎中修改:
	
```html
<!--<a href="{{../dir}}/{{this}}">{{this}}</a>-->
<a href="{{../dir}}/{{file}}">【{{icon}}】{{file}}</a>
```	

# 六、文件压缩

request header 中有 Accept—Encoding：gzip，deflate，告诉浏览器支持的压缩方式

response header 中有 content-Encoding 

最常用文件压缩，gzip等，使用node的内置的zlib模块进行压缩，对于文件是用ReadStream文件流进行读取的，所以对ReadStream进行压缩

```
// helper/compress.js
const  zlib = require('zlib') // 内置压缩文件
module.exports = (readStream, request, response) => {
  const acceptEncoding = request.headers['accept-encoding']
  console.log(acceptEncoding)
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

修改文件的response代码

```
// fs.createReadStream(filePath).pipe(response)
      let readStream = fs.createReadStream(filePath)
      readStream = compress(readStream,request, response)
      readStream.pipe(response)
```

# 七、range 范围请求

# 八、设置缓存

以上的 Node 服务都是浏览器首次请求或无缓存状态下的，那如果浏览器/客户端请求过资源，一个重要的前端优化点就是缓存资源在客户端。**缓存有强缓存和协商缓存**：

强缓存在 Request Header 中的字段是 Expires 和 Cache-Control；

协商缓存在 Request Header 中的字段是：

- If-Modified-Since（对应值为上次 Respond Header 中的 Last-Modified）
- If-None—Match（对应值为上次 Respond Header 中的 Etag）

强缓存使用 Cache-Control，协商缓存使用 If-Modified-Since 或 If-None—Match，如果协商成功则返回 304 状态码