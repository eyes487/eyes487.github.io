---
title: Vue源码解析（三）：数据响应式
categories: 前端
tags: ['Vue','源码','MVVM']
date: 2020-01-28
---

> Vue 最独特的特性之一，是其非侵入性的响应式系统。数据模型仅仅是普通的 JavaScript 对象。而当你修改它们时，视图会进行更新。这使得状态管理非常简单直接，不过理解其工作原理同样重要，这样你可以避开一些常见的问题。

之前写过一篇文章[《实现MVVM原理》](https://blog.eyes487.top/2019/09/10/mvvm-principle.html),今天从源码的角度了解一下数据响应式是如何实现的。在上一篇文章 `initState` 处就是做数据响应式的，今天就从这里开始。

## 1.initState()
core/instance/state.js中
```js
 export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  // 属性初始化
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  // 数据响应式====>从这进入
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
在里面做了一系列数据的初始化，包括props，methods，但是最重要的是data
一般用户都会定义`data`，然后对data做数据响应式，就进入到 `initData`
```js
function initData (vm: Component) {
  // 把数据从options中取出来
  let data = vm.$options.data
  //对data的定义可是函数也可以是对象
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  if (!isPlainObject(data)) {
    data = {}
    process.env.NODE_ENV !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    )
  }
  // proxy data on instance
  // 代理这些数据到实例上，通过实例就可以直接访问到数据
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    //这上面做的是一些判重
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      // 主要代码--代理
      proxy(vm, `_data`, key)
    }
  }
  // observe data
  // 响应式操作
  observe(data, true /* asRootData */)
}
```
这个文件中就是获取data，设置代理，启动响应式observe

## 2.Observer

所有数据响应式的代码都在 `observer` 文件夹下,MVVM框架最重要的就是数据响应式了

```js
export function observe (value: any, asRootData: ?boolean): Observer | void {
  if (!isObject(value) || value instanceof VNode) {
    return
  }

  // 观察者，已经存在直接返回，否则创建新的实例
  let ob: Observer | void
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```
这个方法，主要是判断是否存在(__ob__属性），如果存在就直接返回，如果不存在就通过`new Observer`创建一个，下面看看`Observer`主要做了什么

```js
// 每一个响应式对象都会有一个ob
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor (value: any) {
    this.value = value
    // 为什么在Observer里面声明dep？
    // object里面新增或者删除属性
    // array中有变更方法
    this.dep = new Dep()
    this.vmCount = 0
    // 设置一个__ob__属性引用当前Observer实例
    def(value, '__ob__', this)

    // 判断类型
    if (Array.isArray(value)) {
      // 替换数组对象原型
      if (hasProto) {
        protoAugment(value, arrayMethods)
      } else {
        copyAugment(value, arrayMethods, arrayKeys)
      }
      // 如果数组里面元素是对象还需要做响应化处理
      this.observeArray(value)
    } else {
      // 对象直接处理
      this.walk(value)
    }
  }

  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }

  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```
在Observer构造函数中，
* 设置`_ob_`属性，
* 判断类型是数组还是对象，对象就直接处理，如果是数组就替换数组原型，能改变数组值的只有7个数组方法
* 'push','pop','shift','unshift','splice','sort','reverse'
* 然后遍历对象对每个key值执行 `defineReactive` 方法
* 其次，还额外创建了一个Dep，这是用来管理$set方法增加的属性，通知页面更新，或者数组删除和新增了元素

下面看看defineReacive里做了什么
```js
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  // 和key一一对应
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  // 属性拦截，只要是对象类型均会返回childOb
  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      // 获取key对应的值
      const value = getter ? getter.call(obj) : val
      // 如果存在依赖
      if (Dep.target) {
        // 收集依赖
        dep.depend()
        // 如果存在子ob，子ob也收集这个依赖
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      // 如果新值是对象，也要做响应化
      childOb = !shallow && observe(newVal)
      // 通知更新
      dep.notify()
    }
  })
}
```
目录
[《Vue源码解析（一）：如何解读源码》](https://blog.eyes487.top/2020/01/26/vue-sourcec-analyse-01.html)
[《Vue源码解析（二）：new Vue() 初始化流程》](https://blog.eyes487.top/2020/01/26/vue-sourcec-analyse-02.html)
《Vue源码解析（三）：数据响应式》
