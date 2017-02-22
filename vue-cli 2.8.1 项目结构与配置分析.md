
#  vue-cli 2.8.1 项目源码结构分析

## 项目结构

```
|-- build
|   |-- build.js
|   |-- check-versions.js
|   |-- dev-client.js
|   |-- dev-server.js
|   |-- utils.js
|   |-- vue-loader.conf.js
|   |-- webpack.base.conf.js
|   |-- webpack.dev-conf.js
|   |-- webpack.prod.conf.js
|-- config
|   |-- dev.env.js
|   |-- index.js
|   |-- prod.env.js
|-- src
|   |-- assets
|   |   |--logo.png
|   |-- components
|   |   |--Hello.vue
|   |-- router
|   |   |--index.js
|   |-- App.vue
|   |-- main.js
|-- static
|   |-- .gitkeep
|-- .babelrc
|-- .editorconfig
|-- .gitignore
|-- index.html
|-- package.json
|-- README.md
```

## package.json

每个项目根目录下一般都有一个 `package.json` 文件，其定义了该项目所需的各种模块，
以及项目的配置信息(比如名称、版本、许可证等元数据)。`npm install` 命令根据这个
配置文件，自动下载所需的模块，也就是配置项目所需的运行和开发环境。

### scripts 字段

`scripts` 字段用于运行指定脚本命令。

```
"scripts": {
    "dev": "node build/dev-server.js",
    "build": "node build/build.js"
}
```

当我们执行 `npm run dev` / `npm run build` 时，分别运行了对应的 `value`，即
在终端用 `node` 命令来执行了对应的 `js` 文件，分别对应开发和生产环境。

## webpack 相关配置

参考：[vue-cli#2.0 webpack 配置分析](https://gold.xitu.io/post/584e48b2ac502e006c74a120) 
作者：滴滴公共前端团队 - 王宏宇

以下是个人的一些补充与见解

### dev-server.js

首先是 `dev` 环境

```
// 检查 Node 和 npm 版本
// 引入 check-versions 模块并自执行
require('./check-versions')()

// 获取 config/index.js 的默认配置
// 调用模块时，如果 require() 参数是文件夹的话，node 默认寻找 index.js
var config = require('../config')

// 如果 Node 的环境无法判断当前是 dev / product 环境
// 使用 config.dev.env.NODE_ENV 作为当前的环境
// ↑ NODE_ENV: '"development"'
if (!process.env.NODE_ENV) {
  process.env.NODE_ENV = JSON.parse(config.dev.env.NODE_ENV)
}

// 引入模块

// 一个可以强制打开浏览器并跳转到指定 url 的插件
var opn = require('opn')
var path = require('path')
var express = require('express')
var webpack = require('webpack')
// http-proxy 可以实现转发所有请求代理到后端真实 API 地址，以实现前后端开发完全分离
var proxyMiddleware = require('http-proxy-middleware')
// 使用 dev 环境的 webpack 配置
var webpackConfig = require('./webpack.dev.conf')


// 如果没有指定运行端口，使用 config.dev.port 作为运行端口
// ↑ config 指定了 port: 8080
var port = process.env.PORT || config.dev.port
// 自动打开浏览器，没设置的话就设置为 false
var autoOpenBrowser = !!config.dev.autoOpenBrowser
// 使用 config.dev.proxyTable 的配置作为 proxyTable 的代理配置.
// ↑ cofig 中指定了 proxyTable: {}
// https://github.com/chimurai/http-proxy-middleware
var proxyTable = config.dev.proxyTable

// 启动一个 express 服务
var app = express()
// 启动 webpack 编译配置文件
var compiler = webpack(webpackConfig)

// 启动 webpack-dev-middleware，将 编译后的文件暂存到内存中
var devMiddleware = require('webpack-dev-middleware')(compiler, {
  publicPath: webpackConfig.output.publicPath,
  quiet: true
})

// 启动 webpack-hot-middleware，也就是我们常说的 Hot-reload
var hotMiddleware = require('webpack-hot-middleware')(compiler, {
  log: () => {}
})
// 当 html-webpack-plugin 模板改变是进行页面重新加载
compiler.plugin('compilation', function (compilation) {
  compilation.plugin('html-webpack-plugin-after-emit', function (data, cb) {
    hotMiddleware.publish({ action: 'reload' })
    cb()
  })
})

// 将 proxyTable 中的请求配置挂在到启动的 express 服务上
Object.keys(proxyTable).forEach(function (context) {
  var options = proxyTable[context]
  if (typeof options === 'string') {
    options = { target: options }
  }
  app.use(proxyMiddleware(options.filter || context, options))
})

// 使用 connect-history-api-fallback 匹配资源，如果不匹配就可以重定向到指定地址
app.use(require('connect-history-api-fallback')())

// 将暂存到内存中的 webpack 编译后的文件挂在到 express 服务上
app.use(devMiddleware)

// 将 Hot-reload 挂在到 express 服务上
app.use(hotMiddleware)

// 拼接 static 文件夹的静态资源路径
// ↓ config 中 assetsSubDirectory: 'static', assetsPublicPath: '/',
var staticPath = path.posix.join(config.dev.assetsPublicPath, config.dev.assetsSubDirectory)
// 为静态资源提供响应服务
app.use(staticPath, express.static('./static'))

// express 服务监听的 port
var uri = 'http://localhost:' + port

devMiddleware.waitUntilValid(function () {
  console.log('> Listening at ' + uri + '\n')
})

// 让我们这个 express 服务监听 port 的请求，
// 并且将此服务作为 dev-server.js 的接口暴露
module.exports = app.listen(port, function (err) {
  if (err) {
    console.log(err)
    return
  }

  // 如果不是测试环境，自动打开浏览器并跳到我们的开发地址
  if (autoOpenBrowser && process.env.NODE_ENV !== 'testing') {
    opn(uri)
  }
})
```

