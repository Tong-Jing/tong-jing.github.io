Vue Loader 是一个`Webpack`的 loader，它支持 .vue 单文件组件的各个`<template>`、`<script>` 和 `<style>`以及*自定义语言块*使用非默认语言，比如 style 使用 less/sass 语言，并支持组件中 css scoped 局部作用域和 css module 等。

> 如果在项目开始阶段为前端工程化复杂的配置而困惑，不妨试试使用 IDE 自动构建，推荐使用 WebStorm 新建 Vue 项目，参考 [WebStorm 2018 的破解、汉化与设置使用][1]
 [1]: https://segmentfault.com/a/1190000016007974
 
 
# 一、.vue 单文件组件 (SFC) 规范

.vue 文件是一个自定义的文件类型，用类 HTML 语法描述一个 Vue 组件。每个 .vue 文件包含三种类型的顶级语言块 `<template>`、`<script>` 和 `<style>`，还允许添加可选的自定义块：
 
- `<template>`模板块
	
	一个SFC中最多一个< template >块；其内容将被提取为字符串传递给 vue-template-compiler ，然后 webpack 将其编译为 js 渲染函数，并最终注入到从`<script> `导出的组件中；

- `<script>`脚本块
	
	一个SFC最多一个`<script>`块，会作为一个模块封装。它的默认导出应该是一个 Vue.js 的组件选项对象，也可以导出由 `Vue.extend()` 创建的扩展对象。
	

- `<style>`样式块

	一个 .vue 文件可以包含多个 `<style> `标签，标签可以有 `scoped` 或者 `module` 属性。


- 自定义语言块

	vue-loader 将会使用块名来查找对应的  loader 进行处理。
	
一个 .Vue 文件示例如下：

``` vue
<template src="./template.html">
  <div class="example">{{ msg }}</div>
</template>

<script src="./script.js">
export default {
  data () {
    return {
      msg: 'Hello world!'
    }
  }
}
</script>

<style scoped src="./style.css">
.example {
  color: red;
}
</style>

<custom1>
  This could be e.g. documentation for the component.
</custom1>
```		
	
所有语言块**支持 src 导入**，导入路径遵循和 webpack 模块请求相同的路径解析规则。当 Vue Loader 编译单文件组件中的` <template>` 块时也会将所有遇到的资源 URL 转换为 webpack 模块请求。

> **路径解析规则**：
> 
> * 绝对路径原样保存；
* 以“.”开头，则看做相对模块请求，并根据文件系统上的目录结构解析；
* 以“～”开头，则将其后的作为模块请求，可以引入 node_modules 中的资源；
- 以“@”开头，则其后的内容也被解释为模块请求，“@”在 vue-cli 创建的项目中默认指向"/src"


# 二、Webpack 配置 vue-loader

Vue Loader 和 Webpack 结合，使我们可以使用 .vue 文件来开发 Vue 项目。Vue Loader 的配置和其它的 loader 不太一样。除了通过 rules 规则将 vue-loader 应用到 .vue 的文件上之外，还需要添加 Vue Loader Plugin 插件：

```js
// webpack.config.js
const VueLoaderPlugin = require('vue-loader/lib/plugin')

module.exports = {
  module: {
    rules: [
      // ... 其它规则
      {
        test: /\.vue$/,
        loader: 'vue-loader'
      }
    ]
  },
  plugins: [
    new VueLoaderPlugin()
  ]
}
```
Vue Loader Plugin 的职责是将 .vue 文件中的语言块应用在相应的 rules 上。例如样式匹配 `/\.css$/`的规则，那么它会应用到 .vue 文件里的 `<style>` 块，匹配 `/\.js$/`的规则会应用到 .vue 文件里的 `<script> `块。

所以修改上诉配置示例，使其更加完整：

``` JavaScript
// webpack.config.js
const path = require('path')
const VueLoaderPlugin = require('vue-loader/lib/plugin')

module.exports = {
  mode: 'development',
  module: {
    rules: [
      // ... 其它规则
      {
        test: /\.vue$/,
        loader: 'vue-loader'
      },
      // 它会应用到普通的 `.js` 文件，以及 `.vue` 文件中的 `<script>` 块
      {
        test: /\.js$/,
        loader: 'babel-loader'
      },
      // 它会应用到普通的 `.css` 文件，以及 `.vue` 文件中的 `<style>` 块
      {
        test: /\.css$/,
        use: [
          'vue-style-loader',
          'css-loader'
        ]
      }
    ]
  },
  plugins: [
    // 这个插件使 .vue 中的各类语言块匹配相应的规则
    new VueLoaderPlugin()
  ]
}
```

# 三、Lang 预处理

Vue 支持各类型的**预处理器**来编译语言块，Vue 会根据语言块的`lang`属性和 webpack 配置的 option rules 自动推断相应的 loader。

例如让 css 使用 sass 语言，需要先安装 sass loader 加载器，并在 webpack 配置才可以使用：

```
$ npm install sass-loader node-sass  --save-dev
```

``` javascript
// webpack.config.js -> module.rules
{
    test: /\.sass$/,
    use: [
        'vue-style-loader',
        'css-loader',
        {
            loader: 'sass-loader',
            options: {
                indentedSyntax: true  //sass-loader 默认解析 SCSS 语言
            }
        }
    ]
}
```
在 .vue 文件中的`<style>` 增加 lang 属性并赋值，就可以使用 sass 语法编写 css:

```vue     
<style lang="sass">
/* write SASS here */
  $base-color: #F90;
  article{
	h1 {color: #333}
	p {color: $base-color}
  }
</style>
```

编译后的 style 样式表为：

```
article h1 {color: #333} 
article p {color: #F90}   
```

除了 Vue 默认使用 PostCSS 处理 css，还可以使用 sass，less，stylus 外， 对于 js 部分 Vue 默认使用 babel 处理，还可以用 coffee、typescript 等，具体的 loader & webpack 配置方式见 [Vue Loader 使用预处理器](https://vue-loader.vuejs.org/zh/guide/pre-processors.html)


# 四、CSS Style 的特殊用法

## 4.1 CSS Scoped 局部作用域

当`<style>`标签带 scoped 属性时，创造出 css 的“局部作用域” 只作用于当前组件的元素，类似 Shadow Dom 中的样式封装。可以在同一个组件中使用 scoped 跟 non-scoped styles，如下所示：

```vue
<template>
  <div class="example">hi</div>
</template>

<style>
/* global styles */
</style>

<style scoped>
/* local styles */
.example {
  color: red;
}
</style>

```
通过使用 PostCSS 进行作用域重写，处理后如下：

```
<style>
.example[data-v-f3f3eg9] {
  color: red;
}
</style>

<template>
  <div class="example" data-v-f3f3eg9>hi</div>
</template>
```

当使用 css scoped 时需要注意以下几点：

* **使用 scope 作用域不能弃用 class 或 id 等!**

	考虑到浏览器渲染各种 CSS 选择器的方式，当使用 scoped 时，选择属性选择器如 p { color: red } 在作用域中会慢很多倍（即当与属性选择器组合时）。如果你使用 class 或者 id 代替，比如 .example { color: red }，这样几乎没有性能影响

* **子组件的根节点将受父组件 css scoped 作用域影响**
	
	使用 scope 作用域时，父组件的样式不会泄漏到子组件中。 但子组件的根节点将受父级作用域 CSS 和子级作用域 CSS 的影响。 这是为了父级可以设置子组件根元素的样式以进行布局。
	
* **父组件可以使用‘ >>>  ’或‘ /deep/  ’ 这种深度选择器作用于子组件**

	```vue
	<style scoped>
	.a >>> .b { /* ... */ }
	</style>
	```
	编译为
	
	```
	.a[data-v-f3f3eg9] .b { /* ... */ }
	```
* **通过 v-html 动态创建的 DOM 内容不受 scoped 样式影响**，但可以使用深度选择器进行样式改变
	

> 在 template 中只包含一个外层节点，不能多个节点并列，这个设计思路遵循父组件可以操作子节点的一个根节点，即使在 CSS 局部作用域下依然有效



## 4.2 CSS Modules 模块模式

一个 CSS Module 其实就是一个 CSS 类型的文件，用于模块化和组合 CSS 的系统，在`<style>`上添加 module 特性并配置 vue-loader 的 modules 模式，就可以使用 CSS Modules。

### 4.2.1 CSS Modules 配置

CSS Modules 必须通过向 css-loader 传入 modules: true 来开启

```
// webpack.config.js -> module.rules
{
  test: /\.css$/,
  use: [
    'vue-style-loader',
    {
      loader: 'css-loader',
      options: { 
        modules: true, // 开启 CSS Modules
        localIdentName: '[local]_[hash:base64:8]' // 自定义生成的类名
      }
    }
  ]
}
```


CSS Modules 可以与其它预处理器一起使用，例如`lang="sass"`时的配置如下：

```
// webpack.config.js -> module.rules
{
  test: /\.scss$/,
  use: [
    'vue-style-loader',
    {
      loader: 'css-loader',
      options: { modules: true }
    },
    {
      loader: 'sass-loader',
      options: { indentedSyntax: true }
    }
  ]
}
```



### 4.2.2 CSS Modules 使用

在 .vue 中的 `<style>` 上添加 module 特性，Vue Loader 会将这个 CSS Modules 编译为`$style`计算属性注入到组件中。可以在`<template>` 中通过动态类绑定来使用这个 CSS Modules 局部对象：


```vue
<style module>
.red {
  color: red;
}
</style>

<template>
  <p :class="$style.red">
    This should be red
  </p>
  <p :class="{ [$style.red]: isRed }">
      Am I red?
  </p>
</template>
```
JS 也可以通过 `this.$style`访问 CSS Modules：

```vue
<script>
export default {
  created () {
    console.log(this.$style.red) // "red_1VyoJ-uZ"
  }
}
</script>
```
上述代码中的`this.$style.red`是一个基于文件名和类名生成的标识符。当从 JS 模块导入 CSS 模块时，它会导出包含从本地名称到全局名称的所有映射的一个对象

> 在 CSS Module 中，所有的 url 和 @import 都是被看成模块依赖，例如 url(./image.png) 会被转换为 require(‘./image.png’)，请求的资源可以是在`node_modules`中。


## 4.3 CSS 提取

一些插件可以将 CSS 提取到单独的文件中，为每个包含 CSS 的 JS 文件创建一个 CSS 文件，以便按需加载 CSS 和源代码。

webpack4 使用 mini-css-extract-plugin 插件，配置如下：

```
npm install -D mini-css-extract-plugin
```

只在生产环境下使用 CSS 提取，便于在开发环境下进行热重载。

```
// webpack.config.js
var MiniCssExtractPlugin = require('mini-css-extract-plugin')

module.exports = {
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          {
            loader: MiniCssExtractPlugin.loader,
            options: {
              hmr: process.env.NODE_ENV === 'development',
            },
          },
          'css-loader'
        ]
      }
    ]
  },
  plugins: [
    new MiniCssExtractPlugin({
      filename: '[name].css', // 类似于webpackOptions.output中的相同选项
      chunkFilename: '[id].css'
    })
  ]
}
```

更多配置参考 [github mini-css-extract-plugin](https://github.com/webpack-contrib/mini-css-extract-plugin)

# 五、自定义语言块

除了默认的`<template>`等节点，还可以加自定义节点，并在webpack 配置的 loader 处理自定义语言块；

可以给节点加 lang 属性，此时节点 rule 匹配 lang 的扩展；如果没有标记 lang，则自定义节点的 name 和 rule 需要在 webpack 中声明。

举例自定义一个语言块 `<myblock>`，首先需要自定义 loader 将自定义块注入到组件中，自定义的 myblock-loader 如下：

```js
// myblock-loader.js 
module.exports = function (source, map) {
  this.callback(
    null,
    `export default function (Component) {
      Component.options.__myblock = ${
        JSON.stringify(source)
      }
    }`,
    map
  )
}
```

配置到 webpack

```js
// webpack.config.js -> module.rules
rules: [
	{
	   resourceQuery: /blockType=docs/,
       loader: require.resolve('./docs-loader.js')
    }
]
```
在组件中使用：

```vue
<!-- ComponentB.vue -->
<template>
  <div>Hello</div>
</template>

<myblock>
This is the documentation for component B.
</myblock>
```
组件间复用：

```vue
<!-- ComponentA.vue -->
<template>
  <div>
    <ComponentB/>
    <p>{{ myblock }}</p>
  </div>
</template>

<script>
import ComponentB from './ComponentB.vue';

export default {
  components: { ComponentB },
  data () {
    return {
      myblock: ComponentB.__myblock
    }
  }
}
</script>
```

---
更多内容参考 [Vue Loader 官网](https://vue-loader.vuejs.org/zh/)



