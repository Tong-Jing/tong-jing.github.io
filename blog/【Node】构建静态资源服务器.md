

# 一、准备工作

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


# 二、HTTP 服务

Node 的 http 模块提供 HTTP 服务器和客户端接口，通过 require('http')使用

Node 服务器每次收到 HTTP 请求后都会调用 http.createServer() 这个回调函数

Node 服务器每次收一条请求，都会先解析请求头作为新的 request 的一部分，然后用新的 request 和 respond 对象触发回调函数。

```js
// config/httpConfig.j
module.exports = {
  root: process.cwd(),
  host: '127.0.0.1',
  port: '8877'
}
```

```js

// server/http.js
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

修改 http.createServer()，使用 fs 模块读取客户端请求的静态资源

```
const fs = require('fs')

const server = http.createServer((request, response) => {
  const filepath = path.join(config.root, request.url)
  
  fs.stat(filepath, (err, stats) => {
    if (err) {
      response.statusCode = 404
      response.setHeader('content-type', 'text/plain')
      response.end(`${filepath} is not a file`)
      return;
    }
    if (stats.isFile()) {
      response.statusCode = 200
      response.setHeader('content-type', 'text/plain')
      fs.createReadStream(filepath).pipe(response)
    } else if (stats.isDirectory()) {
      fs.readdir(filepath, (err, files) => {
        response.statusCode = 200
        response.setHeader('content-type', 'text/plain')
        response.end(files.join(','))
      })
    }
  })
})
```

> 成熟的静态资源服务器anywhere，深入理解nodejs作者写的