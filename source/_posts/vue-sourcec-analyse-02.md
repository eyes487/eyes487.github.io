---
title: Vue源码解析（二）：new Vue() 初始化流程
categories: 前端
tags: ['Vue','源码']
date: 2020-01-27
---

上一篇文章已经说到如何来解读框架源码，这篇文章就通过走一遍new Vue()创建一个实例所要经过的过程来解读其中的代码所发挥的作用。
下面不重要的一些代码都会通过`//...`省略掉

## 1.new Vue()

```js
 const app = new Vue({
      el: '#demo',
      data:{
        msg:'hello world'
      }
  })
```
通过断点，我们会进入`instance/index.js`中
```js
function Vue (options) {
  //...
  // 初始化
  this._init(options)
}
initMixin(Vue)  // 通过该方法给Vue添加_init方法
```
很明显，在Vue构造函数中，只执行了`this._init`初始化，进入_init函数，就进入到initMixin中，可见，`_init`是通过下面的initMixin方法添加在函数原型上的。

## 2.initMixin
```js
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // a uid
    vm._uid = uid++

    let startTag, endTag
    // a flag to avoid this being observed
    vm._isVue = true
    // 1.合并选项,默认选项和用户传进来的选项，比如一些全局组件会被合并
    // merge options
    if (options && options._isComponent) {
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }
    /* istanbul ignore else */
    //2. 定义了vm._renderProxy ，后期为render做准备的，作用是在render中将this指向vm._renderProxy
    if (process.env.NODE_ENV !== 'production') {
      initProxy(vm)
    } else {
      vm._renderProxy = vm
    }
    // expose real self
    // 3.
    vm._self = vm
    initLifecycle(vm)  // $parent, $root, $children, $refs
    initEvents(vm)     // 对父组件传入事件添加监听
    initRender(vm)     // 声明$slots,$createElement()
    callHook(vm, 'beforeCreate') // 调用beforeCreate钩子
    initInjections(vm) // 注入数据
    initState(vm)      // 重要：数据初始化，响应式
    initProvide(vm) // 提供数据
    callHook(vm, 'created')

    if (vm.$options.el) {
      vm.$mount(vm.$options.el)
    }
  }
}
```

这个方法主要是为Vue原型上添加`_init`方法，而在_init方法中做的几件事情使我们要关注的。
* `mergeOptions` 合并选项
* 定义了`vm._renderProxy`
* 其他方法(重要)，下面具体介绍

### 2.1 initLifecycle
```js
export function initLifecycle (vm: Component) {
  const options = vm.$options

  // locate first non-abstract parent
  //建立所有组件的父子关系
  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }

  vm.$parent = parent
  vm.$root = parent ? parent.$root : vm

  vm.$children = []
  vm.$refs = {}

  vm._watcher = null
  vm._inactive = null
  vm._directInactive = false
  vm._isMounted = false
  vm._isDestroyed = false
  vm._isBeingDestroyed = false
}
```
在这里是一些跟声明周期相关变量的初始化，比如: 父亲(`$parent`),祖先(`$root`),孩子(`$children`),引用(`$refs`)

### 2.2 initEvents
```js
export function initEvents (vm: Component) {
  vm._events = Object.create(null)
  vm._hasHookEvent = false
  // init parent attached events
  //将父组件模板中注册的事件放到当前组件实例的listeners里面
  const listeners = vm.$options._parentListeners
  if (listeners) {
    updateComponentListeners(vm, listeners)
  }
}
```
处理父组件传入的事件和回调，事件是谁派发谁监听

### 2.3 initRender
```js
export function initRender (vm: Component) {
  vm._vnode = null // the root of the child tree
  vm._staticTrees = null // v-once cached trees
  const options = vm.$options
  const parentVnode = vm.$vnode = options._parentVnode // the placeholder node in parent tree
  const renderContext = parentVnode && parentVnode.context
  vm.$slots = resolveSlots(options._renderChildren, renderContext)
  vm.$scopedSlots = emptyObject
  // 给编译器生成渲染函数使用的，内部使用
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  // render(h) 此处的$createElement就是h
  // 用户render使用的
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

  // $attrs & $listeners are exposed for easier HOC creation.
  // they need to be reactive so that HOCs using them are always updated
  const parentData = parentVnode && parentVnode.data

  /* istanbul ignore else */
  if (process.env.NODE_ENV !== 'production') {
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
    }, true)
    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$listeners is readonly.`, vm)
    }, true)
  } else {
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true)
    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, null, true)
  }
}
```
这个函数中，做的主要的两件事情就是声明 `$slot` ,`$createElement`

### 2.4 callHook

```js
export function callHook (vm: Component, hook: string) {
  // #7573 disable dep collection when invoking lifecycle hooks
  pushTarget()
  const handlers = vm.$options[hook]
  const info = `${hook} hook`
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      invokeWithErrorHandling(handlers[i], vm, null, vm, info)
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook)
  }
  popTarget()
}
```
调用钩子函数，这里调用了`beforeCreate`，之前声明的变量，此时都可以调用了，下面会用此方法调用`created`

### 2.5 initInjections/initProvide
先理解Vue2.0中的Provide/inject
> 这对选项需要一起使用，以允许一个祖先组件向其所有子孙后代注入一个依赖，不论组件层次有多深，并在起上下游关系成立的时间里始终生效。如果你熟悉 React，这与 React 的上下文特性很相似。
```js
//将$options里的provide赋值到当前实例上
export function initProvide (vm: Component) {
  const provide = vm.$options.provide
  if (provide) {
    vm._provided = typeof provide === 'function'
      ? provide.call(vm)
      : provide
  }
}

export function initInjections (vm: Component) {
  const result = resolveInject(vm.$options.inject, vm)  //这个方法主要就是对inject属性中的各个key进行遍历，然后沿着父组件链一直向上查找provide中和inject对应的属性，直到查找到根组件或者找到为止，然后返回结果
  if (result) {
    //对result的一些处理，比如在非生产环境会将result里的值定义成相应式的。
    toggleObserving(false)
    Object.keys(result).forEach(key => {
      /* istanbul ignore else */
      if (process.env.NODE_ENV !== 'production') {
        //...
      } else {
        defineReactive(vm, key, result[key])
      }
    })
    toggleObserving(true)
  }
}
```
注入数据和提供数据，注入数据之后要判重，或者做一些其他的操作，提供给其他组件，所以注入数据在提供数据之前

### 2.6 initState

```js
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  // 属性初始化
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  // 数据响应式
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```
数据响应式，这是很重要的一部分，先大致看看做了什么操作，[详细了解](https://blog.eyes487.top/2019/09/10/vue-sourcec-analyse-03.html)
### 2.7 $mount

下面就执行挂载方法了Vue源码解析（三）：数据响应式
```js
if (vm.$options.el) {
      vm.$mount(vm.$options.el)
}
```
之前提到过，$mount是在`entry-runtime-with-compiler.js`中做了一些额外的判断，但主要的实现方法是在`runtime/index.js`中
```js
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
然后调用`mountComponent()`方法
```js
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode
    if (process.env.NODE_ENV !== 'production') {
      //...
    }
  }
  callHook(vm, 'beforeMount')

  let updateComponent
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    updateComponent = () => {
     //...
    }
  } else {
    // 用户$mount()时，定义updateComponent
    updateComponent = () => {
      vm._update(vm._render(), hydrating)  //会通过new Watcher中的get方法调用
    }
  }

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}
```
vm._update 会调用 vm.render()方法,先执行render方法返回vnode，在通过update转换为真实dom
```js
Vue.prototype._render = function (): VNode {
    const vm: Component = this
    const { render, _parentVnode } = vm.$options 
    //从vm.options中拿到的变量来自于 instance/index.js中给实例添加的属性,通过调用下面这些方法
    //stateMixin(Vue) // $set,$delete,$watch
    // eventsMixin(Vue) // $emit,$on,$off,$once
    // lifecycleMixin(Vue) // _update(),$forceUpdate(),$destroy()
    // renderMixin(Vue)  // _render(), $nextTick

    if (_parentVnode) {
      vm.$scopedSlots = normalizeScopedSlots(
        _parentVnode.data.scopedSlots,
        vm.$slots,
        vm.$scopedSlots
      )
    }

    // set parent vnode. this allows render functions to have access
    // to the data on the placeholder node.
    vm.$vnode = _parentVnode
    // render self
    let vnode
    try {
      currentRenderingInstance = vm
      //  vm.$createElement就是h
      vnode = render.call(vm._renderProxy, vm.$createElement)
    } catch (e) {
      //...
    } finally {
      currentRenderingInstance = null
    }
    // if the returned array contains only a single node, allow it
    if (Array.isArray(vnode) && vnode.length === 1) {
      vnode = vnode[0]
    }
    // return empty vnode in case the render function errored out
    if (!(vnode instanceof VNode)) {
      //...
    }
    // set parent
    vnode.parent = _parentVnode
    return vnode
  }
}
```
```js
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const restoreActiveInstance = setActiveInstance(vm)
    vm._vnode = vnode
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)   
      //这句执行为页面就变为真实dom了，页面数据就发生变化了
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)   
      //diff算法就是在这里发生的，之后会有文章仔细说明
    }
    restoreActiveInstance()  
    // update __vue__ reference
    if (prevEl) {
      prevEl.__vue__ = null
    }
    if (vm.$el) {
      vm.$el.__vue__ = vm
    }
    // if parent is an HOC, update its $el as well
    if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
      vm.$parent.$el = vm.$el
    }
  }
```
## 3.总结
梳理一下创建实例的整个流程
* new Vue(): 调用init
* this._init():   初始化各种属性
* $mount:   调用mountComponent
* mountComponent:   声明updateComponent、创建Watcher
*  _render():   获取虚拟dom  
*  _update():   把虚拟dom转换为真实dom                                                                                





目录
[《Vue源码解析（一）：如何解读源码》](https://blog.eyes487.top/2020/01/26/vue-sourcec-analyse-01.html)
《Vue源码解析（二）：new Vue() 初始化流程》
[《Vue源码解析（三）：数据响应式》](https://blog.eyes487.top/2020/01/26/vue-sourcec-analyse-03.html)
[《Vue源码解析（四）：Vue批量异步更新策略》](https://blog.eyes487.top/2020/01/26/vue-sourcec-analyse-04.html)
