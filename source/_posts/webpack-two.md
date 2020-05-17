---
title: webpack深入学习（二）：进阶篇
categories: 前端
tag: ['webpack','js']
date: 2020-05-17
---

上一篇文章，讲了一些webpack的基础配置，算是对webpack的入门。今天会拓展一些真实项目中可能会用到的配置。


# 一、webpack-dev-server

如果不知道如何安装使用的，请先看[《webpack深入学习（一）：基础配置篇》](https://blog.eyes487.top/2020/05/16/webpack-first.html)

## 1.1 解决跨域
这是我们项目中经常互遇到的一个问题，以前经常都会用到`jsonp`,`cors`来跨域，但是现在，`webpack-dev-server`为我们提供了非常简单的方法，就可以支持跨域了。上一篇文章中，我们提到了`webpack-dev-server`来帮我们启动一个热更新服务器。
```js
//webpack.config.js
module.exports = {
  //...
  devServer: {
    contentBase: path.resolve(__dirname,"./dist")，//把这里当成静态目录了
    open: true, //是都自动打开浏览器窗口
    port: 8080,
    proxy: {
      '/api': 'http://localhost:3000'
    }
  }
};
```
通过上面的配置，我们就可以跨域请求数据了。`http://localhost:3000`是服务器的地址，这相当于一个反向代理，就是代理了服务端的地址，这样我们在浏览器中，是不知道我们向哪个服务器发送的请求，只能看到本机的地址。

## 1.2 本地mock数据

现在项目开发，基本都是前后端分离的，我们写完静态页面之后，可能后端接口还没有写完，我们不能一直就等着后端的接口呀。所以开发之前会给出一个接⼝⽂档，和接⼝联调⽇期的，我们前端就可以本地mock数据，不打断⾃⼰的开发节奏。

`dev-server`给我们提供了两个钩子，分别叫`before`和`after`,表示加载`dev-server `之前和之后要做的事情
```js
module.exports = {
  //...
  devServer: {
    //...
    before(app,server){
        app.get('/api/mock.json',(req,res)=>{
            res.json({
                data: 'hello world'
            })
        })
    },
    after(){
        //...
    }
  }
};
```
这样设置好之后，我们启动这个 `dev-server`服务器，也就是`npm run dev`,然后`http://localhost:8080/api/mock.json`就可以访问了，它会给我们返回`hello world`,这样就可以来模拟后台接口返回数据了。
根据接口文档的格式，先写好请求格式，等到之后联调接口，就只需要替换一下地址就可以啦，是不是很方便呢。。。


## 1.3 Hot Module Replacement (HMR:热模块替换)

之前我们设置`dev-server`,他是通过刷新整个页面，更新代码，那这样的话，我们在页面上做的操作就会没有了，那我们要是想要只更新我们更改的代码应该怎么办呢？？？
`HMR` 就是帮我们做这件事情的，它只会刷新局部，也就是我们更改代码的地方，而我们之前在页面的操作还会保留着。

启动 `hmr` , 在`dev-server`中设置  `hot: true`, 在头部引入 `webpack`, 需要使用 `webpack.HotModuleReplacementPlugin()`
```js
//webpack.config.js
const webpack = require('webpack')

devServer: {
    contentBase: "./dist",
    open: true,
    hot:true
},

plugins： [
    //...
    new webpack.HotModuleReplacementPlugin()
]
```
通过上面的配置，我们试一下，发现浏览器还是刷新了，那要注意了，这里还有一点，需要增加一个 `httpOnly: true`
```js
devServer: {
    contentBase: "./dist",
    open: true,
    hot:true,
    //即便HMR不⽣效，浏览器也不⾃动刷新，就开启hotOnly
    //这为什么会失效呢??它会和MiniCssExtractPlugin.loader有冲突，如何开启了这个css，那么热模块替换就会失效
    hotOnly:true
},
```
上面我们设置了`hotOnly:true`,这样，热模块替换就开启成功了。
**这里有一个注意的地方：** `HMR` 有时会不生效，这是它和 `MiniCssExtractPlugin.loader` 会冲突，这个插件是用来`提取css成单独文件的`，后面在优化部分我们会讲到。。所以，我们一般推荐开发模式下不开启MiniCssExtractPlugin插件，开发模式下也不需要， 在生产模式下才提取css，而且生产模式下也不需要热模块替换了。


上面说的这些，只是`css` 的 HMR，下面我们在看看 `js`的 HMR

** 需要使⽤module.hot.accept来观察模块更新 从⽽更新**
在入口的地方
```js
// index.js
if (module.hot) {
    module.hot.accept("./other.js", function() {
        //....
    });
}
```
在最上层入口，通过监听某个模块，来做一些替换操作。。

这种方式，好像不够高效，那我们平时项目都是怎么做的呢？？？
现在我们开发基本都是使用框架了，react or  Vue, 应该很少有用原生js来写的了。所以社区已经为我们提供了loader来和各种框架和库平滑地交互了。

* React Hot Loader：实时调整 react 组件。
* Vue Loader：此 loader 支持 vue 组件的 HMR，提供开箱即用体验。
* Elm Hot Loader：支持 Elm 编程语言的 HMR。
* Angular HMR：没有必要使用 loader！直接修改 NgModule 主文件就够了，它可以完全控制 HMR API。
想了解更多的朋友，请戳 [这里](https://webpack.docschina.org/guides/hot-module-replacement/#%E5%90%AF%E7%94%A8-hmr)

# 二、Babel 处理 ES6

官⽅⽹站：https://babeljs.io/
中⽂⽹站：https://www.babeljs.cn/
`Babel` 是`JavaScript` 编译器，能将ES6代码转换成ES5代码，让我们开发过程中放⼼使⽤JS新特性⽽不⽤担
⼼兼容性问题。并且还可以通过插件机制根据需求灵活的扩展。
Babel在执⾏编译的过程中，会从项⽬根⽬录下的 `.babelrc` JSON⽂件中读取配置。没有该⽂件会从
loader的options地⽅读取配置。

安装依赖
```js
npm i babel-loader @babel/core @babel/preset-env -D
```
`babel-loader` 是 webpack 与 babel的通信桥梁，不会做把es6转成es5的⼯作，这部分⼯作需要⽤到
`@babel/preset-env` 来做, @babel/preset-env⾥包含了es，6，7，8转es5的转换规则

```js
{
    test: /\.js$/,
    exclude: /node_modules/,
    use: {
        loader: "babel-loader",
        options: {
            presets: ["@babel/preset-env"] 
        } 
    } 
}
```
未完待续...

-------------如果以上内容有不对的地方，还请大家指正------------