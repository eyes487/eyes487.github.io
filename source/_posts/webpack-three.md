---
title: webpack深入学习（三）：优化篇
categories: 前端
tag: ['webpack','js']
date: 2020-05-23
---

前两篇文章，讲解了一些webpack的基本配置，但是随着我们代码越来越多，我们构建的时间也会越来越长，今天就会对webpack做一些优化配置，帮助我们减少构建时间，同时也帮助我们优化输出的代码质量，让代码在生产环境能够更快的让用户访问到。

# 一、缩⼩文件范围 

优化`loader`配置,loader是一个消性能的大户，官方不建议使用过多的loader，但是我们又不能不用
* `test` `include` `exclude` 三个配置项来缩⼩loader的处理范围
* include: 只去哪里查找
* exclude: 排除，不去哪里查找
```js
{
    test: /\.css$/,
    include: path.resolve(__dirname,"./src"), 
}
```

# 二、resolve

## 优化resolve.modules配置

`resolve.modules` 用于配置webpack去哪些目录下寻找第三方模块，默认是['node_modules'],如果没有找到，就会去上一级⽬目录`../node_modules` 找，再没有会去`../../node_modules`中找，以此类推，和Node.js的模块寻找机制很类似.
如果我们的第三方模块都安装在了项目根目录下，就可以直接指明这个路径。
```js
//webpack.config.js
module.exports={
    resolve:{
        modules: [path.resolve(__dirname, "./node_modules")]
    }
}
```

## 优化resolve.alias配置

`resolve.alias` 配置通过别名来将原导⼊路径映射成一个新的导⼊路径
拿`react`为例，我们引⼊的react库，一般存在两套代码
* cjs: 采用`commonJS`规范的模块化代码
* umd: 已经打包好的完整代码，没有采用模块化，可以直接执⾏

默认情况下，`webpack`会从⼊口文件`./node_modules/bin/react/index`开始递归解析和处理依赖的文件。我们可以直接指定文件，避免这处的耗时。使用绝对路径也可以减少构建时间，因为相对路径最后也是转换为绝对路径

```js
//webpack.config.js
module.exports={
    resolve:{
        modules: [path.resolve(__dirname, "./node_modules")],
        alias: {
            react: path.resolve(__dirname,"./node_modules/react/umd/react.production.min.js"  ),
            "react-dom": path.resolve(__dirname,"./node_modules/react-dom/umd/react-dom.production.min.js")
        }
    }
}
```

## 优化resolve.extensions配置

`resolve.extensions`在导⼊语句没带⽂件后缀时，`webpack`会自动带上后缀后，去尝试查找文件是否存在
`extensions`就是来配置这个后缀列表的，webpack默认只支持`js和json` 文件，如果想支持其他的，就可以在这个列表中添加。
```js
//webpack.config.js
module.exports={
    resolve:{
        //...
        extensions: ['.js','.json','.ts',...]
    }
}
```
但是去查找列表需要消耗时间，所以不建议设置很多
* 后缀尝试列表尽量的小
* 导入语句尽量的带上后缀
* 频率高的放在最前面

# 三、externals优化cdn静态资源

前提条件：
* 公司有`cdn` 服务
* 静态资源有部署到cdn 有链接了
* 想使用cdn，因为cdn有很多好处嘛，加快静态文件加载速度等
这样我们在打包的时候，`bundle` ⽂件里，就不用打包进去这个依赖了，体积会更⼩

假如我们使用`jquery`或者`echarts`等，想使用`cdn`资源，在`index.html` 中使用标签引入
```html
<!-- 其他代码 -->
<script src="http://libs.baidu.com/jquery/2.0.0/jquery.min.js"></script>
```
我们希望在使用时，仍然可以通过`import` 的⽅式去引用`(如import $ from 'jquery')`，并且希望`webpack`不会对其进行打包，此时就可以配置`externals`
```js
//webpack.config.js
module.exports={
    //...
    externals: {
        //jquery通过script引入之后，全局中即有了 jQuery 变量
        'jquery': 'jQuery'    
    }
}
```
其实，如果你在`index.html` 中已经引入`jquery` 的资源，可以不用再`import` 中了引入也可以使用了，但是为了`格式统一`，所以在文件中引入了

# 四、publicPath(CDN)

`CDN` 通过将资源部署到世界各地，使得用户可以就近访问资源，加快访问速度。要接入CDN，需要把网页的静态资源上传到CDN服务上，在访问这些资源时，使⽤CDN服务提供的URL。
```js
//webpack.config.js
module.exports={
    //...
    output:{
        publicPath: '//cdnURL.com', //指定存放JS文件的CDN地址
    }
}
```
这样设置之后，在打包出来的文件，在`index.html` 引入的时候，会自动帮我们加上面指定的地址。我们需要把打包出的文件自己上传到cdn服务器上

# 五、MiniCssExtractPlugin 抽离css

之前我们在说到处理`css`的时候，css打包都是直接打包进js文件的，css文件也可以抽离出来单独成一个文件。因为单独生成css,css可以和js并行下载，提高页⾯加载效率

安装依赖
```bash
npm i mini-css-extract-plugin -D
```
```js
const MiniCssExtractPlugin = require("mini-css-extract-plugin");
//webpack.config.js
module.exports={
    module:{
        rules:[
            {
                test: /\.less$/,
                //less-loaser: 会把less转换为css文件
                use: [
                    // "style-loader", //不再需要style-loader，用MiniCssExtractPlugin.loader代替
                    MiniCssExtractPlugin.loader,
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
    plugins: [
        newMiniCssExtractPlugin({
            filename: "css/[name]_[contenthash:6].css",
            chunkFilename: "[id].css"    
        }) 
    ]
}
```

# 六、压缩

## 压缩css
需要使用到两个插件`cssnano `做多方面的的优化，以确保最终生成的文件 对生产环境来说体积是最小的

安装依赖
```bash
npm i cssnano -D 
npm i optimize-css-assets-webpack-plugin -D
```

```js
const OptimizeCSSAssetsPlugin = require("optimize-css-assets-webpack-plugin");

nodule.exports={
    plugins:[
        new OptimizeCSSAssetsPlugin({
            cssProcessor: require("cssnano"), //引⼊入cssnano配置压缩选项
            cssProcessorOptions: {
                discardComments: { 
                    removeAll: true 
                } 
            }
        })
    ]
}
```

## 压缩html
之前我们提到了`html-webpack-plugin` 可以帮助我们创建html模板，引入js文件，它还有很多可配置的属性

```js
module.exports={
    plugins:[
        new htmlWebpackPlugin({      
            title: "测试",      
            template: "./index.html",      
            filename: "index.html",      
            minify: {        
                // 压缩HTML⽂文件        
                removeComments: true, // 移除HTML中的注释        
                collapseWhitespace: true, // 删除空白符与换符        
                minifyCSS: true // 压缩内联css      
            }    
        }),
    ]
}
```

# 七、tree Shaking

`webpack2.x` 开始⽀持 `tree shaking` 概念，顾名思义，`"摇树"`，清除⽆用 `css,js(Dead Code)`
Dead Code 一般具有以下几个特征
* 码不会被执⾏，不可到达
* 代码执⾏的结果不会被用到
* 代码只会影响死变量(只写不读)
* Js tree shaking只⽀持ES module的引入方式！！！！

## Css tree shaking
安装依赖
```bash
npm i glob-all  purify-css purifycss-webpack -D
```
```js
const PurifyCSS=require('purifycss-webpack') 
const glob=require('glob-all')
module.exports={
    //...
    plugins:[
        // 清除⽆用 css
        new PurifyCSS({
            paths: glob.sync([
                // 要做 CSS Tree Shaking 的路径文件
                path.resolve(__dirname, './src/*.html'), // 请注意，我们同样需要对 html 文件进行 tree shaking
                path.resolve(__dirname, './src/*.js')      
            ])    
        })
    ]
}
```
这样，在页面中并没有用到的css就会被摇掉

## Js tree shaking
只支持`import`⽅式引入，不支持`commonjs`的方式引⼊

只需要在配置文件中配置就行，不需要额外的插件
```js
optimization: {
    usedExports: true// 哪些导出的模块被使⽤用了了，再做打包
}
```
只要`mode`是`production`就会生效，develpoment的`tree shaking`是不⽣生效的，因为webpack为了⽅便你的调试
可以查看打包后的代码注释以辨别是否生效。

**生产模式不需要做上面配置，默认开启**

## 副作用

```js
//package.json
"sideEffects":false  //正常对所有模块进行tree shaking  , 仅生产模式有效，需要配合usedExports

//或者在数组面排除不需要tree shaking的模块
"sideEffects":['*.css','@babel/polyfill']
```

# 八、code Splitting

**单页面应用spa:**
打包完成，所有页面只生成一个bundle.js
* 代码体积变大，不利于下载
* 没有合理利用浏览器资源(比如谷歌，可以同时发送6个tcp连接)

**多页面应用mpa:**
如果多个页面引入了一些公共模块，那么可以把这些公共的模块抽离出来，单独打包，公共代码只需要下载一次缓存起来，避免重复下载

```js
module.expports={
    optimization: {    
        splitChunks: {      
            chunks: "all", // 所有的 chunks 代码公共的部分分离出来成为一个单独的文件    
        },  
    }
}
```
splitChunks还有一些其他配置
```js
optimization: {
    splitChunks: {
        chunks: 'async',//对同步 initial，异步 async，所有的模块有效 all
        minSize: 30000,//最⼩尺寸，当模块大于30kb
        maxSize: 0,//对模块进行二次分割时使用，不推荐使用
        minChunks: 1,//打包⽣成的chunk文件最少有几个chunk引用了这个模块
        maxAsyncRequests: 5,//最⼤异步请求数，默认5
        maxInitialRequests: 3,//最大初始化请求书，⼊口文件同步请求，默认3
        automaticNameDelimiter: '-',//打包分割符号
        name: true,//打包后的名称，除了布尔值，还可以接收一个函数function
        cacheGroups: {//缓存组
            vendors: {
                test: /[\\/]node_modules[\\/]/,
                name:"vendor", // 要缓存的分隔出来的 chunk 名称
                priority: -10//缓存组优先级数字越大，优先级越高        
            },
            other:{
                chunks: "initial", // 必须三选一： "initial" | "all" | "async"(默认就是async)
                test: /react|lodash/, // 正则规则验证，如果符合就提取 chunk,
                name:"other",
                minSize: 30000,
                minChunks: 1,        
            },
            default: {
                minChunks: 2,
                priority: -20,
                reuseExistingChunk: true//可设置是否重⽤用该chunk        
            }      
        }    
    }  
}
```

# 九、Scope Hoisting

作⽤域提升`（Scope Hoisting）`是指 webpack 通过 ES6 语法的静态分析，分析出模块之间的依赖关系，尽可能地把模块放到同一个函数中。下⾯通过代码示例来理解
```js
// hello.js
export default'Hello, Webpack';
// index.js
import str from'./hello.js';
console.log(str);
```
打包后，这两个文件会被打包成一个文件

通过配置
```js
module.exports={
    optimization:{
        concatenateModules: true
    }
}
```
这样，`hello.js`和`index.js`就合并成一个函数了，这样打包出来的文件会更小，运行更快

# 十、使用工具量化
`speed-measure-webpack-plugin` 可以测量各个插件和loader所花费的时间

安装依赖
```bash
npm i speed-measure-webpack-plugin -D
```
```js
//webpack.config.js
const SpeedMeasurePlugin=require("speed-measure-webpack-plugin");
const smp=newSpeedMeasurePlugin();
const config= {
    //...webpack配置
}
module.exports=smp.wrap(config);
```

`webpack-bundle-analyzer` 分析webpack打包后的模块依赖关系
安装依赖
```bash
npm i webpack-bundle-analyzer -D
```
```js
const BundleAnalyzerPlugin=require('webpack-bundle-analyzer').BundleAnalyzerPlugin;
module.exports=merge(baseWebpackConfig, {
    //....
    plugins: [
        //...
        new BundleAnalyzerPlugin(),    
    ]
})
```
启动webpack 构建，会默认打开一个窗⼝

# 十一、DLLPlugin 打包插件第三方库，提前编译
Dll动态链接库 ，其实就是做缓存

项⽬中引⼊了很多第三方库，这些库在很长的一段时间内，基本不会更新，打包的时候分开打包来提升打包速度，⽽`DllPlugin` 动态链接库插件，**其原理就是把网页依赖的基础模块抽离出来打包到dll文件中，当需要导⼊的模块存在于某个dll中时，这个模块不再被打包，而是去dll中获取**，这是帮助我们在`开发` 的时候，提升速度，对生产不会有什么影响

动态链接库只需要被编译一次，项目中用到的第三方模块，很稳定，例如react,react-dom，只要没有升级的需求
webpack已经内置了对动态链接库的支持
* `DllPlugin`:⽤于打包出一个单独的动态链接库文件
* `DllReferencePlugin`：⽤于在主要的配置文件中引⼊DllPlugin插件打包好的动态链接库文件

首先，新建一个`webpack.dll.config.js`,这里就用`react`和`react-dom`举例
```js
const path = require("path");
const {DllPlugin} = require("webpack");

module.exports = {
        mode: "development",
        entry: {
            react: ["react", "react-dom"] //! node_modules?  
        },
        output: {
            path: path.resolve(__dirname, "./dll"),
            filename: "[name].dll.js",
            library: "react"  
        },
        plugins: [
            new DllPlugin({
                // manifest.json文件的输出位置
                path: path.join(__dirname, "./dll", "[name]-manifest.json"),
                // 定义打包的公共vendor文件对外暴露的函数名
                name: "react"    
            })  
        ]
}
```
在package.json中添加
```bash
"dev:dll": "webpack --config ./webpack.dll.config.js"
```
然后运行 `npm run dev:dll`
打包完成，你会发现多了一个dll文件，里面有react.dll.js文件，这时已经单独打包出来了
* dll⽂件包含了⼤量模块的代码，这些模块被存放在一个数组里。用数组的索引号为ID,通过变量将自己暴露在全局中，就可以在window.xxx访问到其中的模块
* Manifest.json 描述了与其对应的dll.js包含了哪些模块，以及ID和路路径。

那接下来看看如何使用??
在配置文件中`webpack.dev.config.js`
```js
new DllReferencePlugin({
    manifest: path.resolve(__dirname,"./dll/react-manifest.json")    
})
```
然后还需要在模板文件中，添加链接
```html
<!-- src/index.html -->
<script type="text/javascript" src="../dll/react.dll.js"></script>
```
手动添加使用，体验不好，这里推荐使用`add-asset-html-webpack-plugin`插件帮助我们做这个事情。

安装
```bash
npm i add-asset-html-webpack-plugin -D
```
它会将我们打包后的 dll.js ⽂件注入到我们生成的 index.html 中。在 `webpack.base.config.js` ⽂件中进⾏更改
```js
new AddAssetHtmlWebpackPlugin({
    filepath: path.resolve(__dirname, '../dll/react.dll.js') // 对应的 dll ⽂文件路路径 
})
```

在`webpack5`中，我们使用`HardSourceWebpackPlugin`来做优化,它会缓存在硬件上，和`DLL`相比，一样的优化效果，但是使⽤却及其简单
* 提供中间缓存的作⽤用
* 首次构建没有太大的变化
* 第二次构建时间就会有较大的节省

安装
```bash 
npm i hard-source-webpack-plugin -D
```
```js
const HardSourceWebpackPlugin = require('hard-source-webpack-plugin')
module.exports = {
    plugins: [
        new HardSourceWebpackPlugin()
    ]
}
```

# 十二、happyPack

运行在 Node之上的Webpack是单线程模型的，也就是说Webpack需要⼀个一个地处理任务，不能同时处理多个任务。`Happy Pack`就能让Webpack做到这一点，它将任务分解给多个子进程去并发执行，子进程处理完后再将结果发送给主进程。从而发挥多核 CPU 电脑的威力

安装
```bash
npm i happyPack -D
```
在配置文件中
```js
const os = require('os')
const Happypack = require('happypack')

const happyThreadPool = HappyPack.ThreadPool({ size: os.cpus().length })
//根据操作系统来判断开启几个进程

module.exports = {
    //...
    module:{
        rules:[
            //...
            {
                test: /\.jsx?$/,
                exclude: /node_modules/,
                use: [{ 
                    // 一个loader对应⼀一个id
                    loader: "happypack/loader?id=babel"
                }]
            }, 
            {
                test: /\.css$/,
                include: path.resolve(__dirname, "./src"),
                use: ["happypack/loader?id=css"]
            }
        ],
    },
    plugins:[
        //...
        new HappyPack({
            // ⽤唯一的标识符id，来代表当前的HappyPack是⽤来处理一类特定的文件
            id: 'babel', 
            // 如何处理理.js⽂文件，⽤用法和Loader配置中⼀一样
            loaders: ['babel-loader?cacheDirectory'],
            threadPool: happyThreadPool,
        }), 
        new HappyPack({
            id: "css",
            loaders: ["style-loader", "css-loader"]
        }),
    ]
}
```

`happypack`主要是用来处理loader的，因为它比较费时。在`module`中定义了几个`id`，下面就要创建几个happaypack的实例

**使用happypack不一定会使打包加快，因为开启多线程需要时间，这不适用于小项目，构建时间较久的时候才需要用到这个**

需要注意：
* `happypack`和`mini-css-extract-plugin`一起使用，会有问题,[了解更多](https://github.com/amireh/happypack/issues/242)

