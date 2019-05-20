# 入门配置

### npx webpack

### webpack -h

这条命令会执行node_modules/.bin/ 目录下的可用命令

### webpack 模块可以支持如下:

ES2015 import 语句
CommonJS require() 语句
AMD define 和 require 语句
css/sass/less 文件中的 @import 语句。
样式(url(...))或 HTML 文件`(<img src=...>)`中的图片链接(image 



# module 配置

### module.noParse

防止 webpack 解析那些任何与给定正则表达式相匹配的文件。忽略的文件中不应该含有 import, require, define 的调用，或任何其他导入机制。忽略大型的 library 可以提高构建性能。

```
module: {
    noParse: /jquery|lodash/,
    // 从 webpack 3.0.0 开始,可以使用函数，如下所示
    // noParse: function(content) {
    //   return /jquery|lodash/.test(content);
    // }
  }
```

### module.rules

创建模块时，匹配请求的规则数组。这些规则能够修改模块的创建方式。这些规则能够对模块(module)应用 loader，或者修改解析器(parser)。

加载非JS文件，默认可以处理JS文件，

css-loader: 解析css文件
style-loader：将css style 注入到html节点

use: ['style-loader', 'css-loader']
use 加载器可以链式传递，从右向左进行应用到模块上。

Rule.use 应用于模块指定使用一个 loader。



# CSS 相关配置

### 使用 SASS

```
npm install sass-loader node-sass -D
```

SCSS 是 Sass 3 引入新的语法，其语法完全兼容 CSS3，并且继承了 Sass 的强大功能，由于 SCSS 是 CSS 的扩展，因此，所有在 CSS 中正常工作的代码也能在 SCSS 中正常工作

### Sourse map 源码追踪
sourcemap是为了解决开发代码与实际运行代码不一致时帮助我们debug到原始开发代码的技术，因为webpack会把css，sass模块打包到style模块中，使用sourcemap追踪原始代码，

css-loader和sass-loader都可以通过该 options 设置启用 sourcemap。

```
// webpack.config.js
    rules: [ // 规则数组，修改模块的创建方式
      {
        test: /\.(sc|c|sa)ss$/,  // 正则表达式，处理scss  sass css
        // use: ['style-loader', 'css-loader','sass-loader']
        use: ['style-loader',
          {
          loader: 'css-loader',
          options: {
            sourceMap: true}},
          {
          loader: 'sass-loader',
          options: {
            sourceMap: true}}]
      }
    ]
```

### PostCss 预处理添加前缀

是一个 CSS 的预处理工具，可以帮助我们：给 CSS3 的属性添加前缀(autoprefixer)，样式格式校验（stylelint），提前使用 css 的新特性(postcss-cssnext)，更重要的是可以实现 CSS 的模块化，防止 CSS 样式冲突

```
npm i -D postcss-loader
npm i -D autoprefixer
```

```
rules: [ // 规则数组，修改模块的创建方式
      {
        test: /\.(sc|c|sa)ss$/,  // 正则表达式，处理scss  sass css
        // use: ['style-loader', 'css-loader','sass-loader']
        use: ['style-loader',
          {
            loader: 'css-loader',
            options: {
              sourceMap: true
            }
          },
          {
            loader: "postcss-loader",
            options: {
              ident: 'postcss',
              sourceMap: true,
              plugins: loader => [
                require('autoprefixer')()
              ]
            }
          },
          {
            loader: 'sass-loader',
            options: {
              sourceMap: true
            }
          }
        ]
      }
    ]
```
ident: 'postcss'是唯一标志

## 样式拆分提取

在生产环境下使用样式表抽离成专门的单独文件并且设置版本号， mode: 'production',
webpack4 开始使用： mini-css-extract-plugin插件, 1-3 的版本可以用： extract-text-webpack-plugin

```
npm i -D mini-css-extract-plugin
```

使用 mini-css-extract-plugin 抽取了样式，就不能再用 style-loader。除了修改 module 配置，还需要修改plugins配置。

新建 webpack.product.config.js

```js
// webpack.product.config.js
  module: {
    rules: [ // 规则数组，修改模块的创建方式
      {
        test: /\.(sc|c|sa)ss$/,  // 正则表达式，处理scss  sass css
        use: [
          // 'style-loader', // 不能再用 style-loader注入到 html 中了
          MiniCssExtractPlugin.loader,
  //...        
  plugins: [
    new MiniCssExtractPlugin({
      filename: '[name].css', // 设置最终输出的文件名
      chunkFilename: '[id].css'
    })
  ]
```
其中的name是配置output.filename
并配置pakeage.json 中的script命令，--config 是指定webpack的执行脚本

```
 "scripts": {
    "build": "npx webpack",
    "dist": "npx webpack --config webpack.product.config.js"
  }
```

打包后新增 dist/main.css，而html中的样式失效了，因为没有style注入了，这时需要手动在html中引入main.css 文件


# CSS 和 JS 压缩插件

webpack5 貌似会内置 css 的压缩，webpack4 可以自己设置一个插件（optimize-css-assets-webpack-plugin）

```
npm i -D optimize-css-assets-webpack-plugin
```
webpack4才有minimizer这个配置：

```
// webpack.product.config.js
const OptimizeCSSAssetsPlugin = require('optimize-css-assets-webpack-plugin');

module.exports = {
//...
  optimization: {
    minimizer: [new OptimizeCSSAssetsPlugin({})]
  }
 }
```



## JS 压缩

压缩需要一个插件： uglifyjs-webpack-plugin, 此插件需要一个前提就是：mode: 'production'.

```
npm i -D uglifyjs-webpack-plugin
```

```
// webpack.product.config.js
+ const UglifyJsPlugin = require('uglifyjs-webpack-plugin'); // 压缩js

module.exports = {
  ...
  optimization: {
    minimizer: [
+     new UglifyJsPlugin({ // 压缩 js
        cache: true,
        parallel: true,
        sourceMap: true // set to true if you want JS source maps
      }),
      new OptimizeCSSAssetsPlugin({}) // 压缩 css 文件
    ]
  }
  ···
};
```

这里需要注意的是，如果没有bable兼容ES6语法，则会报错 ERROR Unexpected token等


# 解决 Hash 名

## 配置文件 Hash 名

解决缓存的问题，将文件打版本

```
const path = require('path');
const MiniCssExtractPlugin = require('mini-css-extract-plugin'); // 拆分css style
···
module.exports = {
  mode: 'production',
  entry: './src/index.js',
  output: {
    filename: 'main.[hash].js',
    path: path.resolve(__dirname, './dist')
  },
  ···  
  plugins: [
    new MiniCssExtractPlugin({ // 拆分css
      filename: '[name].[hash].css', // 设置最终输出的文件名
      chunkFilename: '[id].[hash].css'
    })
  ],
  ···  
};
```

修改filename: '[name].[hash].css'

```
> npx webpack --config webpack.product.config.js
Hash: 10c0c7348792960894f6
Version: webpack 4.30.0
```
这时 dist/main.10c0c7348792960894f6.css出现，

## 文件 Hash 名的注入

在前面拆分CSS为单独文件时，HTML是手动引入的，但Hash生成的文件每次都不同，要如何自动引入？

**使用 HtmlWebpackPlugin 插件，可以把打包后的 CSS 或者 JS 文件引用直接注入到 HTML 模板中**，这样就不用每次手动修改文件引用了。

```
npm i -D html-webpack-plugin
```
 HtmlWebpackPlugin是插件，所以在plugins中新增配置：
 
```
+ const HtmlWebpackPlugin = require('html-webpack-plugin');
···
module.exports = {
  mode: 'production',
  ···  
  plugins: [
	···
+   new HtmlWebpackPlugin({
      title: 'Learn webpack', // 默认值：Webpack App
      filename: 'index.html', // 最终生成的文件，默认值： 'index.html'
      template: path.resolve(__dirname, 'src/index.html'), // 模版
      minify: {
        collapseWhitespace: true, // 折叠空白
        removeComments: true, // 移除注释
        removeAttributeQuotes: true // 移除属性的引号
      }
    })
  ],
  ···  
};
```

可以在新建 src/index.html 作为模版，执行打包命令：

```
> npm run dist

> testwebpack@1.0.0 dist /Users/TJing/Documents/webProjects/testWebpack
> npx webpack --config webpack.product.config.js

Hash: b6f4a880e0371e2f8ad3

```

最终打包生成 main.b6f4a880e0371e2f8ad3.css、main.b6f4a880e0371e2f8ad3.js 和 index.html，此时HTML自动注入了CSS和JS：

```html
<!DOCTYPE html>
<html lang=en>
<head>
  <meta charset=UTF-8>
  <title></title>
  <link href=main.b6f4a880e0371e2f8ad3.css rel=stylesheet>
</head>
<body>
<script type=text/javascript src=main.b6f4a880e0371e2f8ad3.js></script>
</body>
</html>
```

## 清理 dist 打包目录

每次编译后 /dist 文件夹都会保存生成的文件，然后就会非常杂乱。
通常，在每次构建前清理 /dist 文件夹，是比较推荐的

[clean-webpack-plugin](https://www.npmjs.com/package/clean-webpack-plugin) 是一个比较普及的管理插件，让我们安装和配置下。

```
npm i -D clean-webpack-plugin
```

```
const CleanWebpackPlugin = require('clean-webpack-plugin');

module.exports = {
  mode: 'production',
···  
  plugins: [
+   new CleanWebpackPlugin()
      ...
  ],
···  
};
```

# 图片处理与优化

## 加载图片

webpack 默认从入口kaishi，一切文件都是模块，通过 module 配置处理各种类型的文件，


如果不加loader

ModuleParseError: Module parse failed: Unexpected character '�' (1:0)
You may need an appropriate loader to handle this file type.


使用file-loader处理文件的导入：

```
npm install --save-dev file-loader
```

```
module.exports = {
  ...
  module: {
    rules: [ 
      ...
	  {
        test: /\.(png|svg|jpg|gif)$/,
        use: ['file-loader']
      }
      ···
    ]  
  } 
}     
```
由于 css 中可能引用到自定义的字体，处理也是跟图片一致。

```
test: /\.(woff|woff2|eot|ttf|otf)$/,
use: [ 'file-loader' ]
```


## 压缩图片

file-loader将图片拷贝到dist目录并更新引用路径，进一步使用 [image-webpack-loader](https://www.npmjs.com/package/image-webpack-loader) 对图片进行压缩和优化，按照 NPM 官网文档进行安装配置：

```
npm install image-webpack-loader --save-dev
```

```
module.exports = {
  ...
  module: {
    rules: [ 
      ...
	  {
        test: /\.(png|svg|jpg|gif)$/,
        use: ['file-loader',
          {
            loader: 'image-webpack-loader',
            options: {
              mozjpeg: {
                progressive: true,
                quality: 65
              },
              // optipng.enabled: false will disable optipng
              optipng: {
                enabled: false,
              },
              pngquant: {
                quality: '65-90',
                speed: 4
              },
              gifsicle: {
                interlaced: false,
              },
              // the webp option will enable WEBP
              webp: {
                quality: 75
              }
            }
          }
        ]
      }
     ···
    ]  
  } 
}
```
原始图片content-length: 557478，大概557K，重新打包压缩后171K

## 小图片处理为 base64 减少 http 请求

url-loader 功能类似于 file-loader，可以把 url 地址对应的文件，打包成 base64 的 Data URI scheme，提高访问的效率。使用base64编码的图片和超链接方式的代码分别如下：

```
<img src="data:image/gif;base64,R0lGODlhAwADAIABAL6+vv///yH5BAEAAAEALAAAAAADAAMAAAIDjA9WADs=" />

<img src="http://gpeng.win/test.png" />
```

对于比较小的图片可以直接打包成 base64 从而减少http请求次数，在最新版的浏览器特别是移动端对base64的兼容性都非常好，可以放心使用。

```
npm install --save-dev url-loader
```

```
module.exports = {
  ...
  module: {
    rules: [ 
      ...
	  {
        test: /\.(png|svg|jpg|gif)$/,
        use: [
          // 'file-loader',
          {
            loader: 'url-loader', // 根据图片大小，把图片优化成base64
            options: {
              limit: 10000 // 10KB
            }
          },
          ···
        ]
      }
      ···
    ]  
  } 
}     
```

# 拆分与合并配置文件

开发环境(development)和生产环境(production)配置文件有很多不同点，但有些配置是公共的，使用 webpack-merge 的工具可以实现两个配置文件进合并，将公共配置抽取到一个配置文件中。

```
npm install --save-dev webpack-merge
```

```
  webpack-demo
  |- package.json
- |- webpack.config.js
+ |- webpack.common.js
+ |- webpack.dev.js
+ |- webpack.prod.js
  |- /dist
  |- /src
    |- index.js
  |- /node_modules
```

一般来说文件的loader都是公共可复用的，代码的拆分与压缩开发环境是不需要的，开发环境需要调试相关的配置。

```
// webpack.common.js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin'); // 将Hash动态生成的文件注入到HTML
const CleanWebpackPlugin = require('clean-webpack-plugin'); // 清理dist目录

module.exports = {
  entry: './src/index.js',
  module: {
    rules: [ // 规则数组，修改模块的创建方式
      ···
    ]
  },
  ···
  plugins: [
    new CleanWebpackPlugin(), // 打包前清理dist
    new HtmlWebpackPlugin({
      title: 'Learn webpack', // 默认值：Webpack App
    })
  ]
};
```

```
//webpack.prod.js
const path = require('path');
const MiniCssExtractPlugin = require('mini-css-extract-plugin'); // 拆分css style
const OptimizeCSSAssetsPlugin = require('optimize-css-assets-webpack-plugin'); // 压缩css
const UglifyJsPlugin = require('uglifyjs-webpack-plugin'); // 压缩js
const merge = require('webpack-merge')
const commonConfig = require('./webpack.common')

module.exports = merge(commonConfig,{
  mode: 'production',
  output: {
    filename: 'main.[hash].js',
    path: path.resolve(__dirname, './dist')
  },
  ···
  plugins: [
    new MiniCssExtractPlugin({ // 拆分css
      filename: '[name].[hash].css', // 设置最终输出的文件名
      chunkFilename: '[id].[hash].css'
    })
  ],
  optimization: {
    minimizer: [
      new UglifyJsPlugin({ // 压缩 JS 文件
        cache: true,
        parallel: true,
        sourceMap: true // set to true if you want JS source maps
      }),
      new OptimizeCSSAssetsPlugin({}) // 压缩 css 文件
    ]
  }
})
```

# JS 配置相关

## js 使用 source map

当 webpack 打包源代码时，可能会很难追踪到错误和警告在源代码中的原始位置。例如，如果将三个源文件（a.js, b.js 和 c.js）打包到一个 bundle（bundle.js）中，而其中一个源文件包含一个错误，那么堆栈跟踪就会简单地指向到 bundle.js。

CSS 可以直接配置 source-map: true，webpack 4.0 新增 devtool: 'inline-source-map' 用于开启 JS 的 source map

```
module.exports = {
  mode: 'development',
  devtool: 'inline-source-map', // js 的 sourcemap
  ···
```

# devServer 开发辅助

## --watch 监控

每次修改完毕后都手动编译太麻烦，最简单解决的办法就是启动 watch 监控更新自动编译。

可以在启动编译时增加 --watch 开启

```
"scripts": {
    "watch": "npx webpack --watch --config webpack.dev.js",
    ···
  },
```
增加 --watch 后，每次修改后手动刷新浏览器页面，就可以看到更新内容。但如何能不刷新页面，自动更新变化呢？

## webpack-dev-server 热更新

使用 [webpack-dev-server](https://www.npmjs.com/package/webpack-dev-server) ，实际就是创建一个简单的 web 服务器，能够实时重新加载(live reloading)。

```
npm install --save-dev webpack-dev-server
```

配置只在开发环境，，除了要配置 devServer 属性，还需要在 plugins 中增加两个插件：

```
const webpack = require('webpack');

module.exports = {
  mode: 'development',
  devtool: 'inline-source-map', // js 的 sourcemap
  devServer: {
    contentBase: './dist',
    hot: true,
    port: 9000
  },
  plugins: [
    new webpack.NamedModulesPlugin(),  // 更容易查看(patch)的依赖
    new webpack.HotModuleReplacementPlugin()  // 替换插件
  ]
 ··· 
```
执行以下命令就会在 http://localhost:9000/ 中看到主页面。注意到，使用 webpack-dev-server 编译后的文件直接在内存中，不会输出到dist目录，但可以使用dist目录中的文件。

```
npx webpack-dev-server --config webpack.dev.js
```

[dev-server官网](https://webpack.docschina.org/configuration/dev-server/) 中其他的配置：
 
```
devServer: {
  clientLogLevel: 'warning', // 可能的值有 none, error, warning 或者 info（默认值)
  hot: true,  // 启用 webpack 的模块热替换特性, 这个需要配合： webpack.HotModuleReplacementPlugin插件
  contentBase:  path.join(__dirname, "dist"), // 告诉服务器从哪里提供内容， 默认情况下，将使用当前工作目录作为提供内容的目录
  compress: true, // 一切服务都启用gzip 压缩
  host: '0.0.0.0', // 指定使用一个 host。默认是 localhost。如果你希望服务器外部可访问 0.0.0.0
  port: 8080, // 端口
  open: true, // 是否打开浏览器
  overlay: {  // 出现错误或者警告的时候，是否覆盖页面线上错误消息。
    warnings: true,
    errors: true
  },
  publicPath: '/', // 此路径下的打包文件可在浏览器中访问。
  proxy: {  // 设置代理
    "/api": {  // 访问api开头的请求，会跳转到  下面的target配置
      target: "http://192.168.0.102:8080",
      pathRewrite: {"^/api" : "/mockjsdata/5/api"}
    }
  },
  quiet: true, // necessary for FriendlyErrorsPlugin. 启用 quiet 后，除了初始启动信息之外的任何内容都不会被打印到控制台。这也意味着来自 webpack 的错误或警告在控制台不可见。
  watchOptions: { // 监视文件相关的控制选项
    poll: true,   // webpack 使用文件系统(file system)获取文件改动的通知。在某些情况下，不会正常工作。例如，当使用 Network File System (NFS) 时。Vagrant 也有很多问题。在这些情况下，请使用轮询. poll: true。当然 poll也可以设置成毫秒数，比如：  poll: 1000
    ignored: /node_modules/, // 忽略监控的文件夹，正则
    aggregateTimeout: 300 // 默认值，当第一个文件更改，会在重新构建前增加延迟
  }
}
```

## webpack 代理服务器


```
npm i -P axios

```

