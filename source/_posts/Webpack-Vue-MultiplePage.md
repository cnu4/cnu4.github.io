---
title: "在多页面项目下使用 Webpack+Vue"
date: 2016-03-21 22:05:51
tags:
	- vue
	- webpack
	- 前端工程
---

## 前言

webpack + vue 能很好的完成单页面应用的开发，官方也提供了很多例子和教程。但使用 webpack 能不能用到多页面项目中，同时又能使用 vue 进行模块组件化开发呢？

这里将结合具体的项目，说明一下我是如何配置的。我们希望能在项目里做到

 - 在每个业务模块下会有很多页面，每个页面都是一个文件夹，所需的资源文件也都放在这个文件夹下
 - 采用 vue + es6 的方式进行组件模块化开发
 - 生成自动引用 webpack 打包好的 js 文件到项目需要的目录
 - 具有良好的开发支持，拥有如 sourseMap，vue 组件的热替换

<!-- more -->

下面 DEMO 的代码地址： [https://github.com/cnu4/Webpack-Vue-MultiplePage](https://github.com/cnu4/Webpack-Vue-MultiplePage)

下面是我们项目的目录结构

## 项目目录结构
```
    ├─Application (thinkphp 配置下的结构,可以结合自己项目做修改)
    │  └─Home
    │      └─View (线上用户访问的.html目录)
    │         └─index (生成的一个模块）
    │             ├─index.html (同一模块的模板放在一个模块目录下)
    │             └─info.html
    ├─Public (线上资源文件目录)
    │  ├─css
    │  ├─imgs
    │  ├─js
    │  └─...
    └─source (前端开发目录)
        ├─another (一个业务模块,每个业务下可能有多个页面)
        │  └─index
        │      ├─app.vue
        │      ├─index.html
        │      ├─index.js
        │      └─static (资源文件)
        ├─components (vue组件目录)
        │  ├─A
        │  │ ├─A.vue
        │  │      
        │  └─B
        │    ├─B.vue
        │          
        └─index (一个业务模块,每个业务下可能有多个页面)
            ├─index
            │  ├─app.vue
            │  ├─index.html
            │  ├─index.js
            │  └─static
            └─info
               └─info.html
```

## 页面文件

每个页面都是一个文件夹，所需的资源文件也都放在这个文件夹下，不需要这个页面时，也只需要删除这个文件夹。

下面是 index 模块下的 index 页面

``` html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>index - Vue Webpack Example</title>
    <!-- webpack 会将入口 JS 文件引入的 CSS 或者 vue 组件中的 css 生成 style 标签或者生成独立的 css 文件并使用 Link 标签加载它 -->
  </head>
  <body>
    <app></app>
    <!-- webpack 的 HtmlWebpackPlugin 插件会根据入口JS文件生成 script 标签并插入在这里或实现按需加载 -->
  </body>
</html>
```

上面是 index 页面的 html 模板，我们无需引入任何 css 和 js ，webpack 会自动帮我打包引入。

其中的 app 标签是我们的 vue 组件，webpac k的加载器会帮我们处理 js 文件中引入的 vue 组件,这样就能正确处理这个标签。

下面 index 页面对应的 js 入口文件

``` js
import Vue from 'vue'
import App from './app'

new Vue({
  el: 'body',
  components: { App }
})
```

## Webpack 配置文件

### 用法

先说下 demo 的运行命令

``` bash 
# 首先安装依赖
npm install

# 开发模式
# 注意非 Windows 环境在 package.json 将开发模式的命令改成：
# NODE_ENV=production webpack
npm run dev

# 打包
npm run build
```

### 配置

下面是 webpack 的配置文件 webpack.config.js，其中用注释指出了关键配置。

``` js
var path = require('path');
var webpack = require('webpack');
// 将样式提取到单独的 css 文件中，而不是打包到 js 文件或使用 style 标签插入在 head 标签中
var ExtractTextPlugin = require('extract-text-webpack-plugin');
// 生成自动引用 js 文件的 HTML
var HtmlWebpackPlugin = require('html-webpack-plugin');
var glob = require('glob');

var entries = getEntry('./source/**/*.js'); // 获得入口 js 文件
var chunks = Object.keys(entries);

module.exports = {
  entry: entries,
  output: {
    path: path.resolve(__dirname, 'Public'), // html,css,js,图片等资源文件的输出路径，将所有资源文件放在 Public 目录
    publicPath: '/Public/',                  // html,css,js,图片等资源文件的 server 上的路径
    filename: 'js/[name].[hash].js',         // 每个入口 js 文件的生成配置
    chunkFilename: 'js/[id].[hash].js'
  },
  resolve: {
    extensions: ['', '.js', '.vue']
  },
  module: {
    loaders: [
      {
        test: /\.css$/,
        // 使用提取 css 文件的插件，能帮我们提取 webpack 中引用的和 vue 组件中使用的样式
        loader: ExtractTextPlugin.extract('style', 'css')
      },
      {
        // vue-loader，加载 vue 组件
        test: /\.vue$/,
        loader: 'vue'
      },
      {
        test: /\.js$/,
        // 使用 es6 开发，这个加载器帮我们处理
        loader: 'babel',
        exclude: /node_modules/
      },
      {
        test: /\.(png|jpg|gif|svg)$/,
        // 图片加载器，较小的图片转成 base64
        loader: 'url',
        query: {
          limit: 10000,
          name: './imgs/[name].[ext]?[hash:7]'
        }
      }
    ]
  },
  babel: {
    presets: ['es2015'],
    plugins: ['transform-runtime']
  },
  vue: { // vue 的配置
    loaders: {
      js: 'babel',
      css: ExtractTextPlugin.extract('vue-style-loader', 'css-loader')
    }
  },
  plugins: [
    // 提取公共模块
    new webpack.optimize.CommonsChunkPlugin({
      name: 'vendors', // 公共模块的名称
      chunks: chunks,  // chunks是需要提取的模块
      minChunks: chunks.length
    }),
    // 配置提取出的样式文件
    new ExtractTextPlugin('css/[name].css')
  ]
};

var prod = process.env.NODE_ENV === 'production';
module.exports.plugins = (module.exports.plugins || []);
if (prod) {
  module.exports.devtool = 'source-map';
  module.exports.plugins = module.exports.plugins.concat([
    // 借鉴 vue 官方的生成环境配置
    new webpack.DefinePlugin({
      'process.env': {
        NODE_ENV: '"production"'
      }
    }),
    new webpack.optimize.UglifyJsPlugin({
      compress: {
        warnings: false
      }
    }),
    new webpack.optimize.OccurenceOrderPlugin()
  ]);
} else {
  module.exports.devtool = 'eval-source-map';
  module.exports.output.publicPath = '/View/';
}

var pages = getEntry('./source/**/*.html');
for (var pathname in pages) {
  // 配置生成的 html 文件，定义路径等
  var conf = {
    // html 文件输出路径
    filename: prod? '../Application/Home/View/' + pathname + '.html' : pathname + '.html',
    template: pages[pathname], // 模板路径
    inject: true,              // js 插入位置
    minify: {
      removeComments: true,
      collapseWhitespace: false
    }
  };
  if (pathname in module.exports.entry) {
    conf.chunks = ['vendors', pathname];
    conf.hash = false;
  }
  // 需要生成几个 html 文件，就配置几个 HtmlWebpackPlugin 对象
  module.exports.plugins.push(new HtmlWebpackPlugin(conf));
}

// 根据项目具体需求，具体可以看上面的项目目录，输出正确的 js 和 html 路径
// 针对不同的需求可以做修改
function getEntry(globPath) {
  var entries = {},
    basename, tmp, pathname;

  glob.sync(globPath).forEach(function (entry) {
    basename = path.basename(entry, path.extname(entry));
    tmp = entry.split('/').splice(-3);
    pathname = tmp.splice(0, 1) + '/' + basename; // 正确输出 js 和 html 的路径
    entries[pathname] = entry;
  });
  console.log(entries);
  return entries;
```

#### 开发模式

运行 `npm run dev` 开发模式运行 demo

根据 webpack 配置文件中 output 的 publicPath 配置项和 HtmlWebpackPlugin 插件的 filename 配置项

> demo 中 dev 环境下中分别是 /View 和pathname + '.html'

所以 demo 中通过 `http://localhost:8080/View/another/index.html` 可以访问到 another 模块下的 index 页面

#### 打包

运行 `npm run build` 打包，可以看到 Application/Home/View 目录下成功生成了按模块分组的 html 文件，这正是项目需要的。

如 Application/Home/View/index 下的 index.html 文件

``` html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>index - Vue Webpack Example</title>
    
  <link href="/Public/css/vendors.css" rel="stylesheet"><link href="/Public/css/index/index.css" rel="stylesheet"></head>
  <body>
    <app></app>
    
  <script src="/Public/js/vendors.91e0fac1fd8493060c99.js"></script><script src="/Public/js/index/index.91e0fac1fd8493060c99.js"></script></body>
</html>
```

venders.css 和 venders.js 文件是 webpack 插件帮我们自动生成的公共样式模块和公共 js 模块。打开页面，还能看到其他资源文件也都被正确处理了。

## 总结

总结一下 webpack 帮我们做了下面几件事

 - 使用 vue-loader 使我们能进行组件化开发。
 - 根据项目需求自动生成按模块分组的 html 文件。
 - 自动提取样式文件，并和打包后的 js 文件加入到自动生成的 html 文件。
 - 将 js 打包为不同的入口文件，并使用插件抽取公用模块。
 - 为开发调试提供需要的环境，包括热替换，sourceMap。


代码地址： [https://github.com/cnu4/Webpack-Vue-MultiplePage](https://github.com/cnu4/Webpack-Vue-MultiplePage)