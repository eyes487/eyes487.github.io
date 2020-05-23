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