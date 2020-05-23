---
title: webpack深入学习（一）：基础配置篇
categories: 前端
tag: ['webpack','js']
date: 2020-05-16
---

平时工作追求效率，一般都会使用脚手架，快速搭建项目，里面webpack都是帮我们配置好了的，导致我很少一段时间都是对webpack一知半解的。最近一段时间就深入了解了一下webpack，所以顺便把它记录下来，可能对其他人也会有点帮助，也方便自己之后回顾。


# 一、什么是webpack

> 本质上，webpack 是一个现代 JavaScript 应用程序的静态模块打包工具。当 webpack 处理应用程序时，它会在内部构建一个 依赖图(dependency graph)，此依赖图会映射项目所需的每个模块，并生成一个或多个 bundle。

![webpack](http://fs.eyes487.top:9999/uploads/1589608486990-webpack.png "图1")

这是官网对webpack的一段解释,上面一张图是，通过webpack处理，把左边的(都可以看成是一个模块)，然后被处理成右边的这种形式。
前端发展很快，出现了很多的新东西，但是浏览器对他们的支持又不是那么好，所以就需要webpack来打包构建等，把他们处理成浏览器支持的东西。

我们先看一下配置项主要会有哪些，先有个印象
```js
module.exports = {
    entry: "./src/index.js", //打包⼊⼝⽂件
    output: "./dist", //输出结构
    mode: "production", //打包环境
    module: {
        rules: [
            //loader模块处理
             {
                test: /\.css$/,
                use: ["style-loader","css-loader"]
             }
         ]
     },
    plugins: [new HtmlWebpackPlugin()] //插件配置
}
```
# 二、安装webpack

**环境准备**

node环境

推荐安装最近版本，**node版本越高，webpack运行速度越快**

**不推荐全局安装**

因为这会将项目找那个的webpack锁定到某个版本，造成不同的项目因为webpack依赖不同而导致冲突，构建失败。
相信大家平时应该都不会去升级全局的版本

**项目安装 推荐**

```bash
# 安装最新的稳定版本
npm i -D webpack

# 安装指定版本
npm i -D webpack@<version>

# 安装最新的体验版本 可能包含bug,不要⽤于⽣产环境
npm i -D webpack@beta

# 安装webpack V4+版本时，需要额外安装webpack-cli
npm i -D webpack-cli
```

**检查安装**
```bash
webpack -v //command not found 默认在全局环境中查找

npx webpack -v// npx帮助我们在项⽬中的node_modules⾥查找

webpack
./node_modules/.bin/webpack -v //到当前的node_modules模块⾥指定webpack
```

# 三、项目初始化

下面，我们就通过项目来一步一步熟悉webpack的配置了

创建一个`webpack-practice`项目，然后通过`npm init -y`快速创建一个配置文件

```bash
mkdir webpack-practice && cd webpack-practice && npm init -y

//安装webpack
npm install webpack webpack-cli -D  //也可以使用yarn
```
我们可以通过`npx webpack -v`检测版本，npx是npm自带的，不用安装，这个命令会帮我们生成一个`软连接`，指向`node_modules`中的`webpack`，之后我们执行打包命令的时候，也会使用这个。

## 3.1 简单实例

创建一个`src/index.js`

```bash
console.log('hello world!!!!!!!!')
```

然后执行命令 `npx webpack`
下面是打包之后输出的信息
![webpack](http://fs.eyes487.top:9999/uploads/1589611109383-webpack2.png "图2")

此时，我们只是写了一个文件，并没有配置任何信息，这竟然也可以打包出来???

这是因为，执行构建的时候，webpack会首先去找`webpack.config.js`,如果没有找到呢，它就会使用自己的默认配置信息。那默认配置信息长什么样呢，下面就来看一看

## 3.2 默认配置
创建一个 `webpack.config.js`

```js
//webpack 是基于nodejs,所以得遵循common.js规范，导出一个对象
const path = require('path')
module.exports={
    //入口
    entry: "./src/index.js",  //默认打包入口
    output: {
        //构建的文件资源放在哪？必须是绝对路径
        path: path.resolve(__dirname,"./dist"),
        //构建的文件资源叫啥？
        filename: "main.s"
    }
}
```

这就是webpack的默认配置了，webpack4⽀持`零配置`使⽤,但是很弱，稍微复杂些的场景都需要额外扩展，下面就来看看更多的配置吧

# 四、配置项

## 4.1 mode

大家是否有看到，上面打包输出，那张图上有一个`warining`,给出了一个警告
大致意思是，必须设置一个`mode`，构建模式，没有设置，就会默认指定生产模式，所以代码会是压缩的
它有下面这些值
![webpack](http://fs.eyes487.top:9999/uploads/1589612165620-webpack-mode.png "图3")

这个值，你可以设置在config.js中，或者在打包的时候写入命令中`npx webpack --mode=production`

## 4.2 context 上下文
这是一个不常用的配置，项目打包的相对路径

## 4.3 entry 入口 / output 出口

三种类型：字符串、数组、对象

字符串已经说过了，就是上面那种格式，下面看看数组

新创建一个文件`src/other.js`
```js
module.exports={
    //入口
    entry: ["./src/index.js", "./src/other.js"]
    output: {
        path: path.resolve(__dirname,"./dist"),
        filename: "index.js"
    }
    //...
}
```
打包完之后的信息
![webpack](http://fs.eyes487.top:9999/uploads/1589614309075-webpack3.png "图4")
依然只输出了一个文件，但是这个文件里包括了两个模块的代码，所以数组形式，是把多个mode打包到一个文件里面

多入口打包，对象形式，一旦有`多入口`的话，就会有`多出口`，所以这里不能指定名称，可以用到`占位符`,无论是一个出口还是多出口，都推荐使用占位符

占位符包括（常用）：
* name： 入口对应的名称
* hash： 整个项目的hash，**每次打包都会创建一个新的**，不利于缓存，每次构建的唯一标识，可以指定长度，[hash:6]
* chunkHash： 根据不同入口entry进行依赖解析，构建对应的chunk，生成相应的hash，**只要组成entry的模块没有内容改动，则对应的hash不变**
```js
module.exports={
    //入口
    entry: {
        index: "./src/index.js",
        other: "./src/other.js"
    },
    output: {
        path: path.resolve(__dirname,"./dist"),
        filename: "[name]-[chunkhash:6].js"
    }
    //...
}
```
下面是打包结果输出
![webpack](http://fs.eyes487.top:9999/uploads/1589618061566-webpack4.png "图5")
左边和右边对比，右边是修改了`other.js`之后的打包信息，可以看出，只有other.js文件的hash发生了改变，这样有利于浏览器缓存


这里，就借机提一下，`bundle`和`chunk`,这里是一个bundle对应一个chunk，bundle就是打包出来的文件，而chunk是代码块，它可以由多个模块组成，这是 webpack 特定的术语被用在内部来管理 building 过程。官网解释，请戳 [这里](https://webpack.docschina.org/glossary)

## 4.4 module

Webpack 默认只⽀持`.json` 和 `.js`模块，不⽀持 不认识其他格式的模块，那么其他格式的模块处理，和处理⽅式就需要loader了

想了解更多，可以去 [官网](https://webpack.docschina.org/loaders/restyle-loader/#src/components/Sidebar/Sidebar.jsx) 查看,常见的loader有:
```bash
style-loader
css-loader
less-loader
sass-loader
ts-loader //将Ts转换成js
babel-loader//转换ES6、7等js新特性语法
file-loader//处理图⽚⼦图
url-loader
eslint-loader
...
```

这些loader，基本都可以见名知义，看名字就知道是处理什么类型文件的。

### 4.4.1 处理样式

#### **style-loader css-loader**

首先安装依赖
```bash
npm install style-loader css-loader -D
```
在js中引入css文件，比如
```css
//index.css
body{
    background: red
}
```
```js
//index.js
import 'index.css'
````

```js
//webpack.config.js
module.exports={
    //入口
    entry:...,
    output: ...,
    module:{
        rules:[
            {
                test: /\.css$/,
                //css-loaser: 是把css模块的类型加入到js模块中区   css in js
                //style-loader: 从js中提取css，在html中创建style标签放入css
                //最好的设计就是，一个loader只做一件事情
                use: ["style-loader","css-loader"]
            }
        ]
    }
    //...
}

```
我们先说一下loader的工作流程：当webpack执行构建的时候，发现了它不认识的模块，然后它就会来module中查找，在`rules`中，通过`后缀名`的形式，配置了处理哪一类文件需要用到什么loader，loader的执行顺序是 `从后往前`,经过处理之后，webpack的构建过程就能顺利地执行下去了

执行打包命令，输出打包文件，我们可以先建一个index.html文件，引入刚才的打包成功的文件，就可以看到css效果了，（下面会讲到通过plugin自动引入打包文件,这里先自己创建一个)

![webpack](http://fs.eyes487.top:9999/uploads/1589622038016-webpack5.png "图6")
从图上可以看到，css通过style标签引入html中了

#### **less-loader**

上面说了css文件，但是我们项目开发一般都会选择，less或者sass，这又该怎么处理呢

安装依赖
```
npm install less less-loader -D
```
```js
//webpack.config.js
module.exports={
    //入口
    entry:...,
    output: ...,
    module:{
        rules:[
            {
                test: /\.less$/,
                //less-loaser: 会把less转换为css文件
                use: ["style-loader","css-loader","less-loader"]
            }
        ]
    }
    //...
}
```
sass跟less处理方式差不多，这里就不说了

**开启css-module**
我们在项目中使用css，一般都是通过css模块化的方式来使用的，也就是通过对象的形式来引入css
```js
//webpack.config.js
module.exports={
    //入口
    entry:...,
    output: ...,
    module:{
        rules:[
            {
                test: /\.less$/,
                //less-loaser: 会把less转换为css文件
                use: ["style-loader",
                {
                    loader: "css-loader",
                    options: {
                        //css modules
                        modules: true
                    }
                },
                "less-loader"]
            }
        ]
    }
    //...
}
```
下面是使用方式：
![webpack](http://fs.eyes487.top:9999/uploads/1589628351472-webpack6.png "图7")

#### **postcss-loader**

它可以帮我们增加一些浏览器前缀，比如一些css的属性，在使用时都需要加上浏览器加上，而postcss-loader就是帮我们做这件事的

```bash
npm install postcss-loader autoprefixer -D
```
```js
module:{
        rules:[
            {
                test: /\.less$/,
                //less-loaser: 会把less转换为css文件
                use: ["style-loader",
                {
                    loader: "css-loader",
                    options: {
                        //css modules
                        modules: true
                    }
                },
                {
                    loader: "postcss-loader",
                },
                "less-loader"]
            }
        ]
    }
```
它要在`css-loader`之前使用，同时还需要创建一个`postcss.config.js`
```js
const autoprefixer = require('autoprefixer')
module.exports = {
    plugins: [autoprefixer({
        //postcss 使用autoprefixer添加前缀的标准
        //last 2 versions: 兼容最近的两个版本
        //>1%: 全球浏览器的时长份额大于1%
        //这两个属性基本可以覆盖普遍浏览器了
        overrideBrowserslist: ["last 2 versions", ">1%"]
    })]
}
```
效果图
![webpack](http://fs.eyes487.top:9999/uploads/1589630109170-webpack7.png "图8")

### 4.4.2 处理图片和文字

#### **file-loader**

安装依赖
```bash
npm install file-loader -D
```
```js
module:{
        rules:[
        //...
            {
                test: /\.(png|jpe?g|gif)$/,
                //less-loaser: 会把less转换为css文件
                use: {
                    loader: "file-loader",
                    options: {
                        name: "[name]-[hash:6].[ext]",
                        outputPath: "images/"
                    }
                },
            }
        ]
    }
```
首先在test中配置图片后缀名，打包的图片文件同样也支持占位符，还有`ext`表示后缀名， `output`是打包图片输出的目录，统一放在images下

同时，这个loader还可以用来处理文字，比如我们经常都会使用到的`字体图标`,在阿里图标中下载几个图标，把`woff2`文件放入`assets`文件下，可以自己随意放在哪里
```js
//webpack.cofig.js
{
    test: /\.(eot|ttf|woff|woff2|svg)$/,
    use: {
        loader:"file-loader",
        options: {
            name: "[name]-[hash:6].[ext]",
        }
    },
    
}
```
```css
//css
@font-face {
    font-family: "iconfont";
    font-display: swap;
    src: url("assets/iconfont.woff2") format("woff2")
}
.iconfont {
      font-family: "iconfont" !important;
      font-size: 30px;
      font-style: normal;
}
```
在页面中引用
```js
//index.js
import styles from './index.less'

let ele = `<div class="${styles.iconfont}">&#xe64d;</div>
<div class="${styles.iconfont}">&#xe652;</div>
<div class="${styles.iconfont}">&#xe64e;</div>`

document.write(ele)
```
效果展示：
![webpack](http://fs.eyes487.top:9999/uploads/1589709745312-webpack8.png "图9")

#### **url-loader**

这是file-loader的`加强版`，它包含了file-loader的全部功能,推荐使用`url-laoder`,这是因为它会支持`limit`

```js
//webpack.cofig.js
{
    test: /\.(png|jpe?g|gif)$/,
    use: {
        loader:"url-loader",
        options: {
            name: "[name]-[hash:6].[ext]",
            outputPath: "images/",
            limit: 1024,  //单位是字节 1024=1kb
        }
    },
    
}
```
上面`limit`的意思是，当图片小于`1kb`的时候，会把图片转为`base64`格式,直接插入代码中，这样就可以减少一些请求，大于1kb就会打包成图片放在images文件夹下。这个具体大小，就看大家怎么衡量了，自己可以设置。一般都推荐小体积的图片资源就转为base64格式


## 4.5 plugins 

插件，它是webpack的一个补充，可以运行在webpack的整个打包过程，每个plugin，都是针对webpack的一个生命周期，做操作的

### **clean-webpack-plugin**

我们上面打包了很多次，然后在dist目录下，生成了很多冗余文件，所以我们需要一个插件来帮我清理掉这些文件

```bash
npm install clean-webpack-plugin -D
```

```js
const {CleanWebpackPlugin} = require('clean-webpack-plugin')
module.exports={
    //入口
    entry:...,
    output: ...,
    module:...,
    plugins: [
        new CleanWebpackPlugin()
    ]
    //...
}
```
这样，它重新打包的时候，就会帮我们清理掉上次打包的文件了。

### **html-webpack-plugin**

上面也说到了，我们需要一个plugin来帮我们自动引用打包文件，不用每次自己创建html，在引用打包的js了。

首先创建一个`src/index.html`

安装依赖
```bash
npm install html-webpack-plugin -D
```
```js
const HtmlWebpackPlugin = require('html-Webpack-Plugin')
// ...
plugins:[
        new HtmlWebpackPlugin({
            template: './src/index.html',
            filename: 'index.html'
        })
    ]
```
这样在打包的时候，就可以把`index.html`也打包进`dist`目录，并且引用打包的js文件了
它还有很多其他的配置项，这里就不一一列举了，感兴趣的可以自己试一下，戳[这里](https://github.com/jantimon/html-webpack-plugin)


上面做了一个webpack的最基本配置，算是对webpack的一些基本了解。下一篇会拓展一些项目中用到的其他配置

-------------如果以上内容有不对的地方，还请大家指正------------