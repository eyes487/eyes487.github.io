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
如果我们的第三⽅方模块都安装在了了项⽬目根⽬目录下，就可以直接指明这个路路径。
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
* cjs: 采⽤用`commonJS`规范的模块化代码
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

# 七、