---
title: webpack打包
date: 2016-09-28 21:06:08
tags: ['webpack']
categories: ['webpack']
---

## webpack特点
1. 可以将所有的静态资源（JS, CSS, Images, font, HTML）当成模块进行打包
2. 自动管理依赖，将按需加载模块进行代码拆分(code splitting)，实际调用时异步加载
3. 社区活跃，有非常多的`plugins`和`loaders`，支持多种规范的加载(ES6，CommonJS，AMD)，配置灵活
4. 支持性能优化，具有开发环境下工具(SourceMaps，development middleware，develop server)

## 使用方式:CLI和API
1. Command Line Interface，命令行用户界面
  全局安装后，在命令行中直接执行webpack。
  ```
    webpack <entry> <output>
  ```
  如果非全局安装，可通过package.json的script配置打包命令。
  webpack的打包参数很多，可以直接通过命令行参数指定，也可以通过配置文件webpack.config.js指定。
  很多配置都是从命令行参数中映射过来的

2. API
  将webpack作为Node.js模块使用
  一般用于和其它构建工具（gulp,grunt等）一起使用

## 安装

  ```javascript
  mkdir webpackTest && cd webpackTest
  npm init
  npm i webpack webpack-dev-server -D
  mkdir src && cd src
  ```

## 命令行相关参数 
### webpack
  
  `webpack` 开发环境下编译
  `webpack -p` 产品编译及压缩
  `webpack --watch` 开发环境下持续的监听文件变动来进行编译(非常快!)
  `webpack -d` 引入 `source maps`
  `webpack --progress` 显示构建进度
  `webpack --display-error-details` 显示打包过程中的出错信息
  `webpack --profile` 输出性能数据，可以看到每一步的耗时
  `webpack --display-modules` 显示隐藏模块
  `webpack --display-chunks` 显示每个chunk下的详细信息

### webpack-dev-server
  一个小型的node.js Express资源服务器。为通过webpack打包生成的资源文件提供Web服务。webpack-dev-server发送关于编译状态的消息到客户端，客户端根据消息作出响应。

  ```
  webpack-dev-server 在 localhost:8080 建立一个 Web 服务器
  webpack-dev-server --devtool eval - 为代码创建源地址
  webpack-dev-server --progress - 显示合并代码进度
  webpack-dev-server --colors - 命令行中显示颜色
  webpack-dev-server --inline 可以自动加上dev-server的管理代码，实现热更新
  webpack-dev-server --hot 开启代码热替换，可以加上HotModuleReplacementPlugin
  webpack-dev-server --port 3000 设置服务端口
  ```

## 配置
### entry
  指定应用的入口点,有3种格式：
  string：指定单个入口文件
  array：多个入口文件，最终打包到一起
  object：多 chunk，key 为 chunk 的 名字，value 指定这 chunk 的入口文件
  string 和 array 最终会打包成一个文件，object 则可以打包出多个文件

### output
  指定打包后的出口文件
```
// 单个文件输出
  module.exports = {
    entry: path.resolve(__dirname, 'src/index.js'), //指定输出文件的文件路径
    output: {
      path: path.resolve(__dirname, 'build'), //指定输出文件的文件名
      filename: 'bundle.js'
    },
    ...
  };
// 多个文件输出
module.exports = {
    entry: {
        a: 'a.js',
        b: 'b.js'
    },
    output: {
        path:  path.resolve(__dirname, 'build'),
        filename: "[name].js"
    }
  ...
};
// 打包库文件,用常规的模块包裹头
module.exports = {
    entry: {
        a: 'a.js'
    },
    output: {
        path: path.resolve(__dirname, 'build'),
        filename: "Lib.js",
        library: 'Lib',
        libraryTarget: 'umd'
    }
  ...
};
//输出的Lib.js文件就有了常规模块化包裹头：
(function webpackUniversalModuleDefinition(root, factory) {
    if(typeof exports === 'object' && typeof module === 'object')
        module.exports = factory();
    else if(typeof define === 'function' && define.amd)
        define([], factory);
    else if(typeof exports === 'object')
        exports["L"] = factory();
    else
        root["L"] = factory();
})(this, function() {
return /******/ (function(modules) { // webpackBootstrap
...
```

### resolve
分析依赖时的路径识别的配置

### externals
  配置外部依赖的，如果使用某些第三方库的时候，就可以通过这个配置来识别外部第三方库的依赖
```
  module.exports = {
    entry: './main.js',
    output: {
      filename: 'L.js',
      library: 'L',
      libraryTarget: 'umd'
    },
    externals: {
      'jquery': {
        root: 'jquery',
        amd: 'jquery',
        commonjs2: 'jquery',
        commonjs: 'jquery'
      }
    }
  };
```
打包后如下
```
(function webpackUniversalModuleDefinition(root, factory) {
    if(typeof exports === 'object' && typeof module === 'object')
        module.exports = factory(require("jquery"));
    else if(typeof define === 'function' && define.amd)
        define(["jquery"], factory);
    else if(typeof exports === 'object')
        exports["Q"] = factory(require("jquery"));
    else
        root["Q"] = factory(root["jQuery"]);
})(this, function(__WEBPACK_EXTERNAL_MODULE_4__) {
return /******/ (function(modules) { // webpackBootstrap
```

### plugins和loaders
1. loaders
  webpack本身只能处理js模块，如果要处理其他类型的文件，就需要使用`loader`进行转换。
  loader是运行在nodejs下的函数，入参是源文件，返回值是转换后的结果，主要用于转换不同类型的资源文件，将他们加载到js文件中。
  多个loader通过管道方式链式调用，每个loader把资源转换成任意格式并传递给下一个`loader`，最后一个`loader`必须返回`JavaScript`。
  常见的loaders: style-loader、css-loader、less-loader、sass-loader、jsx-loader、url-loader、babel-loader、file-loader等
  安装及配置

  ```javascript
    npm i xxx-loader -D

    // webpack.config.js
    module: {
      loaders: [
        {test: /\.js$/, loader: "babel"}, // 通过文件扩展名（或正则表达式）绑定不同类型的文件
        {test: /\.css$/, loader: "style!css"},
        {test: /\.(jpg|png)$/, loader: "url?limit=8192"}, // 可以接受参数，传递配置项给url-loader
        {test: /\.less$/, loader: "style!css!less"}
      ]
    }
  ```
2. plugins
  webpack本身提供了丰富的插件用来满足不同需求，比如commonsPlugin，在打包多个入口文件时会提取出公用的部分，生成common.js
  常用插件：
  UglifyJsPlugin：压缩资源
  CommonsChunkPlugin: 抽取出第三方资源，提取出应用中公共的代码

  ```
  // webpack.config.js
   var openBrowserWebpackPlugin = require('open-browser-webpack-plugin');
   ......
    plugin: [
      new openBrowserWebpackPlugin({ url: 'http://localhost:8080' })
    ]
  ```

  DefinePlugin：定义变量，将变量插件代码吗

  ```
  // webpack.config.js
    ...
    const production = process.env.NODE_ENV ? process.env.NODE_ENV.trim() === 'prod' : false;
    ...
    plugin: [
       new webpack.DefinePlugin({
        __SERVER__: !production,
        __DEVELOPMENT__: !production,
        __DEVTOOLS__: !production,
        'process.env': {
          BABEL_ENV: JSON.stringify(process.env.NODE_ENV),
        },
      }),
    ]

  // package.json
  "scripts": {
      "publish-mac": "export NODE_ENV=prod && webpack-dev-server -p --progress --colors",
      "publish-win": "set NODE_ENV=prod && webpack-dev-server -p --progress --colors"
  }

  // 执行时变量会被替换为ture或false
  if (__DEV__) {
    console.warn('开发环境');
  }
  ...

  ```

  extract-text-webpack-plugin:css文件单独打包
  html-webpack-plugin：解析html模板，生成嵌入了打包资源的index.html页面
  open-browser-webpack-plugin：资源构建完后，自动打开浏览器，提高开发体验
