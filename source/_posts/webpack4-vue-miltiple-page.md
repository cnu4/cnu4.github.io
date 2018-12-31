---
title: Webpack + Vue 多页面项目升级 Webpack 4 以及构建速度优化
date: 2018-12-30 12:15:32
tags:
	- vue
	- webpack
	- 前端工程
---

## 0. 前言

早在 2016 年我就发布过一篇关于在多页面下使用 Webpack + Vue 的配置的文章，当时是我在做自己一个个人项目时遇到了多页面的配置问题，想到别人也可能遇到跟我同样的问题，就把配置的思路分享出来了，[传送门](http://fangxw.me/2016/03/21/Webpack-Vue-MultiplePage/)在这里。

因为那份配置直到现在还有人在关注，同时最近公司帮助项目升级了 Webpack 4，趁机也把之前的配置也升级了一下，而且博客荒废了这么久，都快 9102 年了，不能连年均一篇博文都不到，所以有了下面的分享。

<!-- more -->

下面的配置主要是给在多页面下使用 Webpack 的同学在升级 Webpack 时提供一点思路，多页面的配置思路请点击上面的传送门。

下面代码的地址 [https://github.com/cnu4/Webpack-Vue-MultiplePage](https://github.com/cnu4/Webpack-Vue-MultiplePage)

## 1. Webpack 升级 4.x

### 1.1. 升级和安装相关依赖

 - webpack 升级
 - webpack-cli webapck4.x 需要新加的依赖
 - mini-css-extract-plugin 取代 extract-text-webpack-plugin
 - 其他相关 loader 和 plugin 
   - css-loader
   - file-loader
   - url-loader
   - vue-style-loader
   - vue-template-compiler（注意要保持与 vue 版本一直）
   - html-webpack-plugin@next
 
### 1.2 修改配置

#### mode 构建模式

设置 mode 构建模式，比如 development 会将 process.env.NODE_ENV 的值设为 development

#### mini-css-extract-plugin

删除原 extract-text-webpack-plugin 配置，增加 mini-css-extract-plugin 配置

```js
module.exports = {
  plugins: [
    new  MiniCssExtractPlugin({
      filename:  'css/[name].css'
    }),
  ],
}

module.exports = {
  module: {
    rules: [
      {
        test:/\.vue$/,
        loader: 'vue-loader',
      },
      { test: /\.css$/,
        use: [
          // 开发模式下使用 vue-style-loader，以便使用热重载
          process.env.NODE_ENV !== 'production' ?
            'vue-style-loader' : MiniCssExtractPlugin.loader,
          'css-loader' ] },
    ]
  }
}
```

#### optimization

这是 webpack 4 一个比较大的变动点，webpack 4 中删除了 `webpack.optimize.CommonsChunkPlugin`，并且使用 `optimization` 中的`splitChunk`来替代，下面的配置代替了之前的 CommonsChunkPlugin 配置，同意能提取 JS 和 CSS 文件

```js
module.exports = {
  optimization: {
    splitChunks: {
      vendors: {
        name:  'venders',
        chunks:  'all',
        minChunks: chunks.length
    }
  }
}
```

#### vue-loader 升级

vue-loader 15 注意要配合一个 webpack 插件才能正确使用

```js
const { VueLoaderPlugin } = require('vue-loader') 

module.exports = {
  plugins: [ new VueLoaderPlugin() ]
}

```
#### html-webpack-plugin 升级

升级到 `next`，否则开发下无法正常注入资源文件

#### 文件压缩

 - optimize-css-assets-webpack-plugin
 - terser-webpack-plugin

压缩的配置也移动到了 optimization 选项下，值得注意的是压缩工具换成了 terser-webpack-plugin，这是 webpack 官方也推荐使用的，估计在 webpack 5 中会变成默认的配置，实测打包速度确实变快了很多。

配置

```js
module.exports = {
    minimizer: [
      new TerserPlugin({ // 压缩js
          cache:  true,
          parallel:  true
        }
      }),
      new OptimizeCSSAssetsPlugin({ // 压缩css
        cssProcessorOptions: {
          safe: true
        }
      })
    ]
  }
}
```

## 2. 增加 ES6+ 支持

### 2.1 安装依赖

 - "babel-core": "^6.26.3",
 - "babel-loader": "^7.1.5",
 - "babel-plugin-transform-runtime": "^6.23.0",
 - "babel-preset-env": "^1.7.0",
 - "babel-preset-stage-2": "^6.24.1",
 - "babel-runtime": "^6.26.0",

### 2.2 添加配置文件 .babelrc

```json
{
  "presets": [
    ["env", {
      "modules": false,
      "targets": {
        "browsers": ["> 1%", "last 2 versions", "ie >= 9"]
      },
      "useBuiltIns": "usage"
    }],
    "stage-2"
  ],
  "plugins": ["transform-runtime"]
}
```

### 2.3 增加 webpack 配置

```js
module.exports = {
  modules: {
    rules: [
      {
        test:  /\.js$/,
        loader:  'babel-loader',
        exclude:  /node_modules/
      }
    ]
  }
}
```

### 2.4 更新 eslint 配置

## 3. 打包速度优化

可以使用下面的插件看看打包时间主要耗时在哪

[speed-measure-webpack-plugin](https://github.com/stephencookdev/speed-measure-webpack-plugin)

### 3.1 TerserPlugin 开启 parallel 选项

开启多线程

### 3.2 HappyPack 和 thread-loader 开启 Loader 多进程转换

github 的 Demo 中没有引入，有兴趣的同学可以尝试，在一些耗时的 Loader 确实可以提高速度

```js
const HappyPack = require('happypack');

exports.module = {
  rules: [
    {
      test: /.js$/,
      // 1) replace your original list of loaders with "happypack/loader":
      // loaders: [ 'babel-loader?presets[]=es2015' ],
      use: 'happypack/loader',
      include: [ /* ... */ ],
      exclude: [ /* ... */ ]
    }
  ]
};

exports.plugins = [
  // 2) create the plugin:
  new HappyPack({
    // 3) re-add the loaders you replaced above in #1:
    loaders: [ 'babel-loader?presets[]=es2015' ]
  })
];
```

### 3.3 提前打包公共代码

#### DllPlugin
 
使用 DllPlugn 将 node_modules 或者自己编写的不常变的依赖包打一个 dll 包，提高速度和充分利用缓存。相当于 splitChunks 提取了公共代码，但 DllPlugn 是手动指定了公共代码，提前打包好，免去了后续 webpack 构建时的重新打包。

首先需要增加一个 webpack 配置文件 `webpack.dll.config.js` 专门针对 dll 打包配置，其中用到 `    webpack.DllPlugin`。

执行 `webpack --config build/webpack.dll.config.js` 后，webpack会自动生成 2 个文件，其中**vendor.dll.js** 即合并打包后第三方模块。另外一个 **vendor-mainifest.json** 存储各个模块和所需公用模块的对应关系。

接着修改我们的 webpack 配置文件，在 plugin 配置中增加 `webpack.DllReferencePlugin`，配置中指定上一步生成的 json 文件，然后手动在 html 文件中引用上一步的 **vendor.dll.js** 文件。

后面如果增删 dll 中的依赖包时都需要手动执行上面打包命令来更新 dll 包。下面插件可以自动完成这些操作。

#### AutoDllPlugin

安装依赖 `autodll-webpack-plugin`

AutoDllPlugin 自动同时相当于完成了 DllReferencePlugin 和 DllPlugin 的工作，只需要在我们的 webpack 中添加配置。AutoDllPlugin 会在执行 `npm install / remove / update package-name` 或改变这个插件配件时重新打包 dll。需要注意的是改变 dll 中指定的依赖包不会触发自动重新打包 dll。

实际打包中生成环境是没问题的，但开发模式下在有缓存的情况下，autodll 插件不会生成新的文件，导致 404，所以在 Demo 中暂时关了这个插件。不过 dll 提前打包了公共文件，确实可以提高打包速度，有兴趣的同学可以研究下开发模式下的缓存问题，欢迎在评论中分享。

```js
module.exports.plugins.push(new AutoDllPlugin({
  inject: true, // will inject the DLL bundles to html
  context: path.join(__dirname, '.'),
  filename: '[name].dll.js',
  debug: true,
  inherit: true,
  // path: './',
  plugins: [
    new TerserPlugin({
      cacheL true,
      parallel: true
    }),
    new MiniCssExtractPlugin({
      filename: '[name].css'
    })
  ],
  entry: {
    vendor: ['vue/dist/vue.esm.js', 'axios', 'normalize.css']
  }
}));
```

### 3.4 terser-webpack-plugin

webpack 官方推荐使用的 JS 压缩插件，取代 UglifyJS，大幅提高打包速度

## 4. 其他问题

下面是我公司项目中遇到的问题，各位升级过程中如果遇到同样的问题可以参考一下解决思路。

### 4.1 json-loader

json-loader
webpack4 内置的json-loader 有点兼容性问题，安装 json-loader 依赖和更改配置

解决：

```js
{
  test: /\.json$/,  //用于匹配loaders所处理文件拓展名的正则表达式
  use: 'json-loader', //具体loader的名称
  type: 'javascript/auto',
  exclude: /node_modules/
}
```

### 4.2 vue-loader

vue-loader 升级到 15.x 后，会导致旧的 commonjs 写法加载有问题，需要使用 `require('com.vue').default` 的方式引用组件

13的版本还可以设置 esModule，14 以后的版本不能设置了，vue 文件导出的模块一定是 esModule

解决：使用 `require('com.vue').default` 或者 `import` 的方式引用组件

[esModule option stopped working in version 14 · Issue #1172 · vuejs/vue-loader · GitHub](https://github.com/vuejs/vue-loader/issues/1172)

尤大大建议可以自己写一个 babel 插件，遇到 require vue 文件的时候自动加上 default 属性，这样就不用改动所有代码，我们在项目中也是这样处理的。

### 4.3 提取公共 css 代码

scss 中 import 的代码不能被提取到公共 css 中

解决：改到 js 中引入就可以，详见下面 issue

[mini-css-extract-plugin + sass-loader + splitChunks · Issue #49](https://github.com/webpack-contrib/mini-css-extract-plugin/issues/49)

### 4.4 mini-css-extract-plugin filename 不支持函数

mini-css-extract-plugin 的 filename 选项不支持函数，所以只能转用其他方式解决

解决：使用插件 FileManagerPlugin 在构建后移动文件，等 filename 支持函数后再优化

[feat: allow the option filename to be a function · Issue #143 · webpack-contrib/mini-css-extract-plugin · GitHub](https://github.com/webpack-contrib/mini-css-extract-plugin/issues/143)

