
#  vue-cli 2.8.1 项目源码结构分析

vue-cli 2.8.1 已经支持 webpack 2.2

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

```js
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

以下是大部分引用和个人的一些补充

### dev-server.js

首先是 `dev` 环境


```js
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

### webpack.dev.conf.js

```js
var utils = require('./utils')
var webpack = require('webpack')
var config = require('../config')
// webpack 配置合并插件
var merge = require('webpack-merge')
// base 配置
var baseWebpackConfig = require('./webpack.base.conf')
// 使用 html-webpack-plugin 插件，这个插件可以帮我们自动生成 html 并且注入
// 到 .html 文件中
var HtmlWebpackPlugin = require('html-webpack-plugin')
var FriendlyErrorsPlugin = require('friendly-errors-webpack-plugin')

// 将 Hol-reload 相对路径添加到 webpack.base.conf 的 对应 entry 前
Object.keys(baseWebpackConfig.entry).forEach(function (name) {
  baseWebpackConfig.entry[name] = ['./build/dev-client'].concat(baseWebpackConfig.entry[name])
})

// 将我们 webpack.dev.conf.js 的配置和 webpack.base.conf.js 的配置合并
module.exports = merge(baseWebpackConfig, {
  module: {
    // 使用 styleLoaders
    rules: utils.styleLoaders({ sourceMap: config.dev.cssSourceMap })
  },
  // 配置开发工具
  devtool: '#cheap-module-eval-source-map',
  plugins: [
    // definePlugin 接收字符串插入到代码当中, 所以你需要的话可以写上 JS 的字符串
    // 编译时配置全局变量
    new webpack.DefinePlugin({
      'process.env': config.dev.env
    }),
    // HotModule 插件在页面进行变更的时候只会重绘对应的页面模块，不会重绘整个 html 文件
    new webpack.HotModuleReplacementPlugin(),
    // 使用了 NoErrorsPlugin 后页面中的报错不会阻塞，但是会在编译结束后报错
    new webpack.NoEmitOnErrorsPlugin(),
    // 将 index.html 作为入口，注入 html 代码后生成 index.html文件
    new HtmlWebpackPlugin({
      filename: 'index.html',
      template: 'index.html',
      inject: true
    }),
    new FriendlyErrorsPlugin()
  ]
})
```

### webpack.base.conf.js

webpack.dev.conf.js 中又引入了 webpack.base.conf.js，先来看 base 是怎样的

```js
var path = require('path')
// 引入小工具
var utils = require('./utils')
var config = require('../config')
// vue-loader 的配置
var vueLoaderConfig = require('./vue-loader.conf')

// 拼接我们的工作区路径为一个绝对路径
function resolve (dir) {
  return path.join(__dirname, '..', dir)
}

module.exports = {
  entry: {
    // 编译入口文件
    app: './src/main.js' 
  },
  output: {
    // 编译的输出根路径
    // ↓ config 中 assetsRoot: path.resolve(__dirname, '../dist')
    path: config.build.assetsRoot,
    // 编译输出的文件名
    filename: '[name].js',
    // 正式发布环境下编译输出的发布路径
    publicPath: process.env.NODE_ENV === 'production'
      ? config.build.assetsPublicPath
      : config.dev.assetsPublicPath
  },
  resolve: {
    // 自动补全的扩展名
    extensions: ['.js', '.vue', '.json'],
    // 2.2 更新，告诉 webpack 目录解析模块时应搜索什么。
    modules: [
      resolve('src'),
      resolve('node_modules')
    ],
    // 默认路径代理，例如 import Vue from 'vue'，会自动到 'vue/dist/vue.common.js'中寻找
    alias: {
      'vue$': 'vue/dist/vue.common.js',
      'src': resolve('src'),
      'assets': resolve('src/assets'),
      'components': resolve('src/components')
    }
  },
  module: {
    // 各种不同类型文件的 loader 的配置
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: vueLoaderConfig
      },
      {
        test: /\.js$/,
        // 用 babel 转码
        loader: 'babel-loader',
        include: [resolve('src'), resolve('test')]
      },
      {
        test: /\.(png|jpe?g|gif|svg)(\?.*)?$/,
        loader: 'url-loader',
        query: {
          limit: 10000,
          name: utils.assetsPath('img/[name].[hash:7].[ext]')
        }
      },
      {
        test: /\.(woff2?|eot|ttf|otf)(\?.*)?$/,
        loader: 'url-loader',
        query: {
          limit: 10000,
          name: utils.assetsPath('fonts/[name].[hash:7].[ext]')
        }
      }
    ]
  }
}

```


### config/index.js

index.js 文件中有 dev 和 prod 两种环境的配置参数

```js
...

module.exports = {
  // 生产环境
  build: {
    // 使用 config/prod.env.js 中定义的编译环境
    env: require('./prod.env'),
    // 编译输出的 html 文件
    index: path.resolve(__dirname, '../dist/index.html'),
    // 编译输出的静态资源根目录
    assetsRoot: path.resolve(__dirname, '../dist'),
    // 输出的二级目录
    assetsSubDirectory: 'static',
    // 编译发布上线路径的根目录，可配置为资源服务器域名或 CDN 域名
    assetsPublicPath: '/',
    // 是否开启 cssSourceMap
    productionSourceMap: true,
    // 是否开启 gzip
    // 用前确保 npm install --save-dev compression-webpack-plugin
    productionGzip: false,
    // 需要使用 gzip 压缩的文件扩展名
    productionGzipExtensions: ['js', 'css'],
    // 运行完成执行的命令
    bundleAnalyzerReport: process.env.npm_config_report
  },
  // 开发环境
  dev: {
    // 使用 config/dev.env.js 中定义的编译环境
    env: require('./dev.env'),
    // 运行测试页面的端口
    port: 8080,
    // 是否自动打开浏览器
    autoOpenBrowser: true,
    assetsSubDirectory: 'static',
    assetsPublicPath: '/',
    // 需要 proxyTable 代理的接口（可跨域）
    proxyTable: {},
    // 是否开启 cssSourceMap
    cssSourceMap: false
  }
}
```

### vue-loader.conf.js

[文档地址](http://vue-loader.vuejs.org/en/)

```js
var utils = require('./utils')
var config = require('../config')
var isProduction = process.env.NODE_ENV === 'production'

module.exports = {
  // 通过 utils.cssLoaders() 返回需要的 loaders
  // 方法有判断环境以返回不同的配置，譬如是否压缩 css，是否生成 sourceMap 等
  // 包括处理 .vue 文件里 <style lang=""></style> 里不同的预处理样式编码
  // 详细查看 utils.js
  loaders: utils.cssLoaders({
    sourceMap: isProduction
      ? config.build.productionSourceMap
      : config.dev.cssSourceMap,
    extract: isProduction
  }),
  postcss: [
    // 用 autoprefixer 插件处理
    require('autoprefixer')({
      browsers: ['last 2 versions']
    })
  ]
}
```

然后就到 `build` 命令

### build.js

```js
require('./check-versions')()

process.env.NODE_ENV = 'production'

// 一个很好看的 loading 插件
var ora = require('ora')
var rm = require('rimraf')
var path = require('path')
// 给终端显示代码加点颜色
var chalk = require('chalk')
var webpack = require('webpack')
var config = require('../config')
// 引入生产环境的配置
var webpackConfig = require('./webpack.prod.conf')

// 使用 ora 打印出 loading 信息
var spinner = ora('building for production...')
spinner.start()

// rm 是一个执行 rm -rf 的模块
// 这里应该是递归删除文件夹，再创建并拷贝资源的操作
rm(path.join(config.build.assetsRoot, config.build.assetsSubDirectory), err => {
  if (err) throw err
  // 开始 webpack 编译
  webpack(webpackConfig, function (err, stats) {
    // 成功的回调函数
    spinner.stop()
    if (err) throw err
    process.stdout.write(stats.toString({
      colors: true,
      modules: false,
      children: false,
      chunks: false,
      chunkModules: false
    }) + '\n\n')

    console.log(chalk.cyan('  Build complete.\n'))
    // 要使用 http 代码，否则无法浏览
    console.log(chalk.yellow(
      '  Tip: built files are meant to be served over an HTTP server.\n' +
      '  Opening index.html over file:// won\'t work.\n'
    ))
  })
})
```

### webpack.prod.conf.js

```js
var path = require('path')
var utils = require('./utils')
var webpack = require('webpack')
var config = require('../config')
var merge = require('webpack-merge')
var baseWebpackConfig = require('./webpack.base.conf')
// 一个复制文件目录的插件
var CopyWebpackPlugin = require('copy-webpack-plugin')
// 一个可以插入 html 并且创建新的 .html 文件的插件
var HtmlWebpackPlugin = require('html-webpack-plugin')
// 一个 webpack 扩展，可以提取一些代码并且将它们和文件分离开
// 如果我们想将 webpack 打包成一个文件 css js 分离开，那我们需要这个插件
var ExtractTextPlugin = require('extract-text-webpack-plugin')
var OptimizeCSSPlugin = require('optimize-css-assets-webpack-plugin')

var env = config.build.env

// 合并 webpack 配置文件
var webpackConfig = merge(baseWebpackConfig, {
  module: {
    // 使用的 loader
    rules: utils.styleLoaders({
      sourceMap: config.build.productionSourceMap,
      extract: true
    })
  },
  devtool: config.build.productionSourceMap ? '#source-map' : false,
  output: {
    // 编译输出目录
    path: config.build.assetsRoot,
    // 编译输出文件名
    // 我们可以在 hash 后加 :6 决定使用几位 hash 值
    filename: utils.assetsPath('js/[name].[chunkhash].js'),
    // 没有指定输出名的文件输出的文件名
    chunkFilename: utils.assetsPath('js/[id].[chunkhash].js')
  },
  plugins: [
    // http://vuejs.github.io/vue-loader/en/workflow/production.html
    new webpack.DefinePlugin({
      'process.env': env
    }),
    // 压缩代码的插件
    new webpack.optimize.UglifyJsPlugin({
      compress: {
        warnings: false
      },
      sourceMap: true
    }),
    // 将 css 文件分离出来
    new ExtractTextPlugin({
      filename: utils.assetsPath('css/[name].[contenthash].css')
    }),
    // Compress extracted CSS. We are using this plugin so that possible
    // duplicated CSS from different components can be deduped.
    new OptimizeCSSPlugin(),
    // generate dist index.html with correct asset hash for caching.
    // you can customize output by editing /index.html
    // see https://github.com/ampedandwired/html-webpack-plugin
    // 输入输出的 .html 文件
    new HtmlWebpackPlugin({
      filename: config.build.index,
      template: 'index.html',
      // 是否注入 html
      inject: true,
      // 压缩的方式
      minify: {
        removeComments: true,
        collapseWhitespace: true,
        removeAttributeQuotes: true
        // more options:
        // https://github.com/kangax/html-minifier#options-quick-reference
      },
      // necessary to consistently work with multiple chunks via CommonsChunkPlugin
      chunksSortMode: 'dependency'
    }),
    // split vendor js into its own file
    // 没有指定输出文件名的文件输出的静态文件名
    new webpack.optimize.CommonsChunkPlugin({
      name: 'vendor',
      minChunks: function (module, count) {
        // any required modules inside node_modules are extracted to vendor
        return (
          module.resource &&
          /\.js$/.test(module.resource) &&
          module.resource.indexOf(
            path.join(__dirname, '../node_modules')
          ) === 0
        )
      }
    }),
    // extract webpack runtime and module manifest to its own file in order to
    // prevent vendor hash from being updated whenever app bundle is updated
    // 没有指定输出文件名的文件输出的静态文件名
    new webpack.optimize.CommonsChunkPlugin({
      name: 'manifest',
      chunks: ['vendor']
    }),
    // copy custom static assets
    new CopyWebpackPlugin([
      {
        from: path.resolve(__dirname, '../static'),
        to: config.build.assetsSubDirectory,
        ignore: ['.*']
      }
    ])
  ]
})

// 开启 gzip 的情况下使用下方的配置
if (config.build.productionGzip) {
  // 加载 compression-webpack-plugin 插件
  var CompressionWebpackPlugin = require('compression-webpack-plugin')
  
  // 向webpackconfig.plugins中加入下方的插件
  webpackConfig.plugins.push(
    // 使用 compression-webpack-plugin 插件进行压缩
    new CompressionWebpackPlugin({
      asset: '[path].gz[query]',
      algorithm: 'gzip',
      test: new RegExp(
        '\\.(' +
        config.build.productionGzipExtensions.join('|') +
        ')$'
      ),
      threshold: 10240,
      minRatio: 0.8
    })
  )
}

if (config.build.bundleAnalyzerReport) {
  var BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin
  webpackConfig.plugins.push(new BundleAnalyzerPlugin())
}

module.exports = webpackConfig
```


以上，查看 webpack(v2.2) 文档或了解插件接口可以移步：

- [api](https://webpack.js.org/api/)
- [configuration](https://webpack.js.org/configuration/)
- [plugins](https://webpack.js.org/plugins/)
- [loaders](https://webpack.js.org/loaders/)

## 其他

### .babelrc

```js
{
  // 预设转码规则
  "presets": [
    ["es2015", { "modules": false }],
    "stage-2"
  ],
  // 转码插件
  "plugins": ["transform-runtime"],
  "comments": false,
  // 针对特定环境变量进行不同的操作
  "env": {
    "test": {
      "plugins": [ "istanbul" ]
    }
  }
}

```

[Babel 入门教程](http://www.ruanyifeng.com/blog/2016/01/babel.html) 作者：阮一峰

### .editorconfig

项目编码规范文件


以上。


