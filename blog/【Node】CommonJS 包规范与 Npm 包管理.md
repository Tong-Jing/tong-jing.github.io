NPM 实践了 CommonJS 包规范规范，帮助我们安装和管理依赖包，使得 Node 项目的第三方模块更加规范便捷，可以在 [NPM 平台](https://www.npmjs.com/)上找到所有共享的插件。

# 一、CommonJS 包规范

CommonJS 包规范的定义分为两部分：用于组织文件目录的**包结构**和用于描述包信息的**包描述文件 package.json**。

## 1.1 包结构
一个包由相当于一个存档文件，可压缩为`.zip` 或`tar.gz`，安装时解压还原。完全符合 CommonJS 包规范的目录包含以下文件：

```
|-- .bin  // 存放可执行的二进制文件
|-- lib   // 存放 Javascript 文件
|-- doc   // 存放文档
|-- test  // 存放单元测试用例
|-- package.json  // 包描述文件

```

## 1.2 包描述文件

包描述文件 package.json 是一个 JSON 文件，在包的根目录下。NPM 的所有行为都与 package.json 中的字段有关，Node 程序的依赖项也体现在这些字段上。

CommonJS 包规范定义了 package.json 中的字段，NPM 实现时对 CommonJS 包规范中的字段也进行取舍和新增，常用字段有：

- name: 包名称；
- description: 包描述；
- keyword: 关键字数组，用于 [NPM](https://www.npmjs.com/) 分类搜索；
- **repository**: 代码托管位置列表；
- homepage: 当前包的网址；
- bugs: 反馈 bug 的邮箱或网址；
- **dependencies**: 使用当前包所需要的依赖包列表，NPM 根据这个属性自动加载依赖；
- **devDependencies**: 后续开发时需要安装的依赖包列表；
- **main**: 模块入口，使用`require()`引入时优先检查该字段。如果 main 字段不存在，Node 按照模块文件定位的规则依次查找包目录下的 index.js、index.node、index.json；
- **scripts**: 包管理器用来安装、编译、测试包的命令对象。
- **bin**: 配置包的 bin 字段后，可以通过`npm install package_name -g`将包添加到执行路径中，之后可以“全局使用”。



# 二、npm 管理

[NPM](https://www.npmjs.com/) 帮助 Node 完成第三方模块的发布、安装和依赖。可以直接执行`$ npm` 查看所有命令。

使用`$ npm init` 可以快速生成一个 package.json 文件。

## 2.1 npm install 原理
使用 `npm install`安装依赖包是 NPM 最常用的功能，例如执行`npm install express`后，npm 向 **registry** 查询模块压缩包的网址，下载压缩包后 NPM 会在当前的 `node_module` 目录下创建 express 目录，将包解压还原在此。

> **registry** 是 NPM 模块仓库提供了一个查询服务，例如 npmjs.org 的查询服务网址 https://registry.npmjs.org/ ，加模块名 https://registry.npmjs.org/vue 就得到包含 Vue 模块所有版本的信息 JSON 对象，也可以使用`$ npm view vue`查询。

Node 项目使用`require('express')`引入 express 模块时，`require()`方法在路径分析时按照**模块路径查找策略**，沿当前路径向上逐级查找`node_module`目录，最终定位到 express 目录。

> 包的安装和模块引入是相辅相成的， 可以进一步理解 [Node 模块加载原理](https://segmentfault.com/a/1190000018424385#articleHeader1)

## 2.2 npm install 使用
 `npm install`默认将包和 package.json 的依赖关系保存在`dependencies`，但在可以通过一些额外的标志来控制它们的保存位置和方式：
 
- `-P` or `--save` or `--save-prod`: 依赖在`dependencies`，默认值.

- `-D` or ` --save-dev`: 依赖在 `devDependencies`.

- `-O` or `--save-optional`: 依赖在 `optionalDependencies`.

- `--no-save`: 防止包依赖保存在 `dependencies`.

例如 `npm install express -D` 就会将 express 依赖关系保存在`devDependencies`。

在`npm install`一个模块时经常纠结要安装在`devDependencies`还是`dependencies`，从字面意思看前者用于生产环境，后者用于开发环境。

在官方的定义中，如果环境变量 NODE_ENV 设置为 production，执行 `npm install --production` 时 npm 会默认安装`dependencies`里面的依赖项，不会去安装`devDependencies`里的。并且推荐`dependencies`里配置正式运行时必须依赖的插件，`devDependencies`通常用来放我们开发或测试的工具，比如 Webpack，Gulp，babel，eslint等。

在实际开发过程中，Node 包的安装是依据 require/import 模块机制，无论是`-P`还是`-D`指令都会把依赖下载到 `node_modules `文件夹，`-P`还是`-D`只是修改了`dependencies`对象，在安装这个库进行开发调试的时候，可以通过`npm install`一键安装这两个目录下所有的依赖。

## 2.3 全局安装

使用 `-g`或 `--global`可以将包安装为“全局可用”，但需要注意的是，**全局安装并不意味将模块包安装为一个全局包**，也不是可以在任何地方都可以`require()`引入。
  
  实际上`-g`命令是将模块包安装在“全局”的`node_module`中，即 Node 可执行文件相同的路径下，并通过配置 bin 字段链接。
  
  例如使用命令行查看 Node 可执行文件的位置：
  
  ```
  $ which node
  /usr/local/bin/node
  ```
  那么全局安装模块的实际位置就是`/usr/local/lib/node_modules`（*在 Finder 中用 command+shift+G 快捷键访问隐藏目录* ）
  
  
---
  
进一步了解 NPM 的使用可以看 [NPM DOCS](https://docs.npmjs.com/)，NPM更多命令 [NPM CLI](https://docs.npmjs.com/cli-documentation/)

---
继续加油哦永远十八岁的少女～  






## NPM 实现过程，木易杨总结的