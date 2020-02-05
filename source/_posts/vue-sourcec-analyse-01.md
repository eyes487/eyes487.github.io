---
title: Vue源码解析（一）：如何解读源码
categories: 前端
tags: ['Vue','源码']
date: 2020-01-26
---

以前听人说解读一个框架的源码，最好的方法就是自己写一个小实例，通过浏览器断点（f10，应该大家都知道的）看运行这个实例执行了哪些步骤，想知道某个方法是如何实现的，就执行某个方法，通过断点查看它到底做了些什么。这个方法是很有用的，今天就通过此方法向大家介绍解读源码的步骤。

Vue版本: 2.6.11

## 1.准备工作

### 1.1 克隆源码
```bash
地址： https://github.com/vuejs/vue
通过命令： git clone https://github.com/vuejs/vue.git
当前版本： 2.6.10
```

### 1.2 安装依赖
```bash
安装依赖： cd vue && npm install 
打包工具rollup： npm install rollup -g  (如果以前没有安装)
```

### 1.3 修改dev脚本
在`package.json`中 在scripts中找到dev，添加 --sourcemap，如下
```bash
 "scripts": {
    "dev": "rollup -w -c scripts/config.js --sourcemap --environment TARGET:web-full-dev",
    "dev:cjs": "rollup -w -c scripts/config.js --environment TARGET:web-runtime-cjs-dev",
```
文件打包之后都是一个压缩文件，通过sourcemap可以定位源文件

### 1.4 执行
```js
npm run dev
```
打包之后，会在dist文件夹下新增一个vue.js.map文件，vue.js也会被修改
这样准备工作就算完成了

## 2.目录结构
下面是一些核心文件，以及他们的用途

![目录结构](http://fs.eyes487.top:9999/uploads/1580133474890-vue-category.png "图1")
```bash
dist- - -发布目录，所有输出文件都在里面
      （打包出的文件，关键字含义）
       runtime- - -仅包含运行时，不包含编译器
       common- - -cjs规范，用于webpack1
       esm- - -ES模块，用于webpack2+
       umd(什么都不带的)- - -兼容cjs和amd，一般用于浏览器
```
## 3.查找入口文件

### 3.1 在`package.json`中，dev命令中
```bash
"dev": "rollup -w -c scripts/config.js --sourcemap --environment TARGET:web-full-dev",
```
代码里可以发现两个关键点 `scripts/config.js`和`web-full-dev`,所以去到config.js可以发现
```js
// Runtime+compiler development build (Browser)
  'web-full-dev': {
    entry: resolve('web/entry-runtime-with-compiler.js'),  //<=====这个文件
    dest: resolve('dist/vue.js'),
    format: 'umd',
    env: 'development',
    alias: { he: './entity-decoder' },
    banner
  },
```
### 3.2 web/entry-runtime-with-compiler.js
这个文件位于 `src/platforms/web/`下面

**如果遇到`/* istanbul ignore if */`,这个代码不重要，甚至可以直接删掉，只用于调试阶段，下面我贴的源码就直接删掉以节约空间了,只会贴出主要代码,重要部分都会写上注释**
```js
import Vue from './runtime/index'
...
// 保存原来的$mount
const mount = Vue.prototype.$mount
// 覆盖默认$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && query(el)
  // 解析option
  const options = this.$options
  // resolve template/el and convert to render function
  if (!options.render) {
    let template = options.template
    // 模板解析 
    if (template) {
      if (typeof template === 'string') {
        if (template.charAt(0) === '#') {
          template = idToTemplate(template)
        }
      } else if (template.nodeType) {
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
    } else if (el) {
      template = getOuterHTML(el)
    }
    // 如果存在模板，执行编译
    if (template) {
      // 编译得到渲染函数
      const { render, staticRenderFns } = compileToFunctions(template, {
        outputSourceRange: process.env.NODE_ENV !== 'production',
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns
    }
  }
  // 执行挂载
  return mount.call(this, el, hydrating)
}
function getOuterHTML (el: Element): string {
  if (el.outerHTML) {
    return el.outerHTML
  } else {
    const container = document.createElement('div')
    container.appendChild(el.cloneNode(true))
    return container.innerHTML
  }
}

Vue.compile = compileToFunctions
```

* 从代码可以看出，new Vue的时候，优先级是render，template，el，最终都是得到render函数
* 这个文件的主要作用：覆盖$mount,执行模板解析和编译工作
* 这个文件发现Vue引入自`runtime/index`

### 3.3 runtime/index.js
```js
import Vue from 'core/index'
import { extend, noop } from 'shared/util'
import { mountComponent } from 'core/instance/lifecycle'

// install platform patch function
// 指定补丁方法：传入虚拟dom转换为真实dom
Vue.prototype.__patch__ = inBrowser ? patch : noop

// public mount method
// 实现$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  // 初始化，将首次渲染结果替换el
  return mountComponent(this, el, hydrating)
}
```
* 这个文件主要定义$mount方法和其他额外配置
* 这个文件发现Vue引入自`core/index`

### 3.4 core/index.js
```js
import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'

// 定义全局api
initGlobalAPI(Vue)

```

* 这个文件主要定义全局api，在Vue山挂在一些其他方法
* 这个文件发现Vue引入自`instance/index`

### 3.5 instance/index.js
```js
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'

// 构造函数
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  // 初始化
  this._init(options)
}

initMixin(Vue)  // 通过该方法给Vue添加_init方法
stateMixin(Vue) // $set,$delete,$watch
eventsMixin(Vue) // $emit,$on,$off,$once
lifecycleMixin(Vue) // _update(),$forceUpdate(),$destroy()
renderMixin(Vue)  // _render(), $nextTick

export default Vue
```

* 终于在这个文件中找到了Vue构造函数，里面只执行了`init`方法，init方法是通过initMixin()给Vue添加_init方法
* 同时，也添加了很多实例方法

在init.js文件中initMixin()方法
```js
// 主要代码
    vm._self = vm
    initLifecycle(vm)  // $parent, $root, $children, $refs
    initEvents(vm)     // 对父组件传入事件添加监听
    initRender(vm)     // 声明$slots,$createElement()
    callHook(vm, 'beforeCreate') // 调用beforeCreate钩子
    initInjections(vm) // 注入数据
    initState(vm)      // 重要：数据初始化，响应式
    initProvide(vm) // 提供数据
    callHook(vm, 'created')
```

**这次主要是寻找入口文件，所以没有对里面的方法去进行深究，之后会有专门的文章对他们进行解读**

## 4. 实例
在examples中，建一个test.html
```html
<head>
    <script src="../../dist/vue.js"></script>
</head>
<div id="demo">
    <p>{{msg}}</p>
        
</div>
<script>
    // 创建实例
    const app = new Vue({
        el: '#demo',
        // template: '<div>template</div>',
        // render(h){return h('div','render')},
        data:{msg:'hello world!!!!!'}
    })
</script>

```
就可以通过断点在浏览器中看到执行的过程了，如果想看某个方法具体执行了什么，就进入该方法查看，从开始到浏览器渲染完成，就是整个渲染流程所执行的过程。
![Vue调试步骤](http://fs.eyes487.top:9999/uploads/1580122844391-vue-tiaoshi.png "图2")

Vue源码解读方法，差不多就是这样了，之后会有具体文章分析Vue的重要实现过程。
学习源码，知道大概思路，设计思想就行了，不用太深究其细节，框架也是一个产品，里面会包含大量业务代码。
框架中，基本都是使用英文注释，对英文不太好的朋友会不太友好，`vscode`中推荐使用`comment Translate`这个插件，可以把注释翻译成中文。

目录
《Vue源码解析（一）：如何解读源码》
[《Vue源码解析（二）：new Vue() 初始化流程》](https://blog.eyes487.top/2020/01/26/vue-sourcec-analyse-02.html)
[《Vue源码解析（三）：数据响应式》](https://blog.eyes487.top/2020/01/26/vue-sourcec-analyse-03.html)
[《Vue源码解析（四）：Vue批量异步更新策略》](https://blog.eyes487.top/2020/01/26/vue-sourcec-analyse-04.html)
[《Vue源码解析（五）：虚拟dom和diff算法》](https://blog.eyes487.top/2020/01/26/vue-sourcec-analyse-05.html)