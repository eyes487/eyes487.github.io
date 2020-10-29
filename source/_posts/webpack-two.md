---
title: webpack深入学习（二）：进阶篇
categories: 前端
tag: ['webpack','js']
date: 2020-05-17
---

上一篇文章，讲了一些webpack的基础配置，算是对webpack的入门。今天会拓展一些真实项目中可能会用到的配置。如果不知道如何做基础配置的，请先看[《webpack深入学习（一）：基础配置篇》](https://blog.eyes487.top/2020/05/16/webpack-first.html)


# 一、webpack-dev-server

上一篇文章中，我们是使用npx webpack进行打包，每修改一次文件，都需要通过这个命令打包，这也太不方便了吧。所以webpack为我们提供了一个插件`webpack-dev-server`

## 1.1 webpack-dev-server 启动项目

首先安装依赖
```bash
npm i webpack-dev-server -D
```
然后在package.json中设置
```js
"scripts": {
    //...
    "dev": "webpack-dev-server"
  },
```
启动项目
```bash
npm run dev
```
看到下面这个提示，就说明已经启动成功了，在`http://localhost:8080/`
![webpack](https://www.eyes487.top/fs/uploads/1589713844017-webpack9.png "图1")
这样之后修改页面，它就会自动帮我们刷新了。这时，会发现，dist目录下被清空了，因为现在都是通过内存来读取文件，这样反应速度会更快了。还可以在`devserver`中做一些配置项
```js
//webpack.config.js
module.exports={
    //...
    devserver:{
        contentBase: path.resolve(__dirname,"./dist")，//把这里当成静态目录了
        open: true, //是都自动打开浏览器窗口
        port: 8080,
        //...
    }
}
```

## 1.2 解决跨域
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

## 1.3 本地mock数据

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


## 1.4 Hot Module Replacement (HMR:热模块替换)

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
通过上面的配置，我们试一下，发现浏览器还是刷新了，那要注意了，这里还有一点，需要增加一个 `hotOnly: true`
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

**需要使⽤module.hot.accept来观察模块更新 从⽽更新**
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

不了解babel的朋友，可以查看刘小夕小姐姐的这篇文章，[《不容错过的 Babel7 知识》](https://juejin.im/post/5ddff3abe51d4502d56bd143)

`Babel` 是`JavaScript` 编译器，能将`ES6`代码转换成`ES5`代码，让我们开发过程中放⼼使⽤JS新特性⽽不⽤担⼼兼容性问题。并且还可以通过插件机制根据需求灵活的扩展。

`Babel`在执⾏编译的过程中，会从项⽬根⽬录下的 `.babelrc` ⽂件中读取配置。没有该⽂件会从
`loader的options` 地⽅读取配置。

## babel-loader

安装依赖
```js
npm i babel-loader @babel/core @babel/preset-env -D
```

`@babel/core` 是babel的核心，babel的功能都依靠它
`babel-loader` 是 webpack 与 babel的通信桥梁，不会做把es6转成es5的⼯作，这部分⼯作需要⽤到
`@babel/preset-env` 来做, @babel/preset-env⾥包含了es，6，7，8转es5的转换规则

```js
{
    test: /\.js$/,
    exclude: /node_modules/,  //排除node_modules中的文件
    use: {
        loader: "babel-loader",  
        options: {
            presets: ["@babel/preset-env"] 
        } 
    } 
}
```
通过上面的方式，就可以把ES6+语法转换为ES5了。但是你可能会发现一个问题，假如你的代码中有`promise`，或者`async/await`,它并没有被转换，对于那些不支持promise的浏览器，这就是一个麻烦的地方了，这样我们又可以引出一个新的东西`polyfill`(垫片)，可以理解为ES6+的ECMA规范库

## @babel/polyfill

首先安装依赖，这个依赖并不是只有开发的时候需要使用，在生产的时候也需要使用

```bash
npm i @babel/polyfill -S
```
然后在入口文件出引入这个文件,所有代码之前
```js
//index.js
import "@babel/polyfill"

//...
```
与此同时，我们会遇到一个问题，这个库中我们又很多方法并没有使用到，把整个库都引入的话，会让我们的打包文件，体积增大很多。那我们需要解决这个问题，给垫片瘦身，实现按需加载，减少冗余。polyfill的初衷就是，我缺少什么，你就给我垫上。我们可以通过`useBuiltIns`字段来设置
修改webpack.config.js
```js
options: {
    presets: [            
        ["@babel/preset-env",{
            targets: {
                // edge: "17",配置浏览器的版本
                // firefox: "60",
                // chrome: "67",
                // safari: "11.1"                
            },
            corejs: 2,//新版本需要指定核⼼心库版本
            useBuiltIns: "entry"//按需注⼊入              
            }            
        ]          
    ]        
}
```
`useBuiltIns` 选项是`babel 7` 的新功能，这个选项告诉`babel`如何配置`@babel/polyfill`。
它有三个参数可以使⽤用：
* entry: 需要在webpack的⼊口文件里`import "@babel/polyfill"`一次。babel会根据你的使用情况导⼊垫片，没有使用的功能不会被导⼊相应的垫片。
* usage: 不需要import，全自动检测，但是要安装@babel/polyfill。（试验阶段）
* false: 如果你`import"@babel/polyfill"`，它不会排除掉没有使⽤的垫⽚片，程序体积会庞⼤大。(不推荐)

**这个polyfill，有一个缺点就是，它会污染全局对象，因为都是直接挂在window上的，比如在开发UI组件的时候。**


## @babel/plugin-transform-runtime

当我们开发的是组件库，⼯工具库这些场景的时候，`polyfill`就不不适合了了，因为`polyfill`是注⼊入到全局变量量，window下的，`会污染全局环境`，所以推荐闭包⽅方式：`@babel/plugin-transform-runtime`，它不不会造成全局污染

安装依赖
```bash
npm i @babel/plugin-transform-runtime -D
npm i @babel/runtime-corejs3 -S
```
修改配置⽂文件：注释掉之前的`presets`，添加`plugins`
```js
options: {
    presets: [            
        ["@babel/preset-env",{    
            }            
        ]          
    ],
    plugins: [    
            ["@babel/plugin-transform-runtime",
                {
                   "corejs": 3,
                }    
            ]  
        ]      
}
```
它也是支持按需加载的，他不会造成全局污染，它不会挂载到window上，它通过替换的形式，假如缺少promise，他会创建一个_promise来替换。 更多配置，可以查看[这里](https://babeljs.io/docs/en/babel-plugin-transform-runtime#docsNav)

## @babel/preset-react
我们 在开发react项目的时候，又是如何来解析jsx语法的呢??babel也为我们提供了一个插件
```bash
npm i @babel/preset-react -D
```
在配置文件中加上
```js
options: {
    presets: [            
        ["@babel/preset-env",
            //...       
        ],
        ["@babel/preset-react"]         
    ]        
}
```
这个插件就是来做语法解析,这样我们就可以使用jsx语法开发react项目了。当然，如果你不想使用js文件，也可以创建jsx文件，把配置文件后缀名改为`test: /\.jsx$/`就行了。

如果上面`babel`文件配置太多，会让`js`文件显得很庞大。所以我们可以把babel的配置移到它自己的配置文件里`.babelrc`。


# 三、Sourcemap

源代码与打包后的代码的映射关系，通过sourceMap定位到源代码。
在开发模式下，是默认开启sourcemap的，所以我们平时开发的时候，出现了错误，都会直接定位到源文件。那生产模式又是怎么配置的呢?

```js
//webpack.config.js
module.exports = {
    //...
    devtool: 'source-map'， //cheap-eval-source-map  ...
}
```
开启这个选项之后，它会生成一个对应的 `.map`文件，里面表示打包文件和源代码的关系映射。

它还有很多可以其他可以选择的模式，每个模式的构建速度和结果不同，信息越详细，构建速度越慢，所以需要权衡这两个方面。
如果想了解更多选项，请戳[这里](https://www.webpackjs.com/configuration/devtool/)

生产环境是不建议开启sourcemap的，但是有一些特殊场景，比如要做错误解析，需要开启sourcemap的话，也不要把map文件上传到公网上，这样比较安全。


# 四、dev和prod模式 区分打包

把所有配置都放在一个配置文件中，会显得很乱，不利于管理可读性很差，所以我们可以对不同环境做一下区分，在通过`webpack-merge` 来合并配置

三个配置文件
* `webpack.base.config.js` 基础配置
* `webpack.dev.config.js`  开发独有的配置
* `webpack.prod.config.js` 生产独有的配置

安装这个插件
```bash
npm i webpack-merge -D
```

配置文件
```js
//webpack.dev.config.js
const merge = require('webpack-merge')
const baseConfig = require("./webpack.base.config.js")

const devConfig = {
    //...
}
module.exports = merge(baseConfig,devConfig)
```

在`package.json`中配置脚本
```js
"scripts":{

    "dev":"webpack-dev-server --config ./webpack.dev.config.js",
    "build":"webpack --config ./webpack.prod.config.js"
}
```
之后执行 `npm run dev`或者`npm run build` 来分别打包了

## 基于环境变量区分
平时我们在代码中，可能也需要针对生产或者是开发环境，做一些配置，那如何获取到当前的环境呢???

安装
```bash
npm i cross-env -D
```
`cross-env`这个插件，是用来抹平各个操作系统之间的差异，比如windows和mac它们的使用路径的方式可能是不一样的

在package.json中设置
```js
"dev": "cross-env NODE_ENV=dev webpack --config ./webpack.dev.config.js",
```
那我们在代码中就可以拿到
```js
process.env.NODE_ENV  //dev
```

-------------如果以上内容有不对的地方，还请大家指正------------