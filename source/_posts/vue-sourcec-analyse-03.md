---
title: Vue源码解析（三）：数据响应式
categories: 前端
tags: ['Vue','源码','MVVM']
date: 2020-01-28
---

> Vue 最独特的特性之一，是其非侵入性的响应式系统。数据模型仅仅是普通的 JavaScript 对象。而当你修改它们时，视图会进行更新。这使得状态管理非常简单直接，不过理解其工作原理同样重要，这样你可以避开一些常见的问题。

之前写过一篇文章[《实现MVVM原理》](https://blog.eyes487.top/2019/09/10/mvvm-principle.html),今天从源码的角度了解一下数据响应式是如何实现的。在上一篇文章 `initState` 处就是做数据响应式的，今天就从这里开始。

Vue版本: 2.6.11

## 1.initState()
src/core/instance/state.js中
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
// src/core/instance/state.js中
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
// src/core/observer/index.js
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
// observer/index.js
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
* `push`,`pop`,`shift`,`unshift`,`splice`,`sort`,`reverse`
* 然后遍历对象对每个key值执行 `defineReactive` 方法
* 其次，还额外创建了一个Dep，这是用来管理$set方法增加的属性，通知页面更新，或者数组删除和新增了元素

先看看数组覆盖的原型是如何做的
```js
//src/core/observer/array.js
// 获取数组原型
const arrayProto = Array.prototype
// 备份
export const arrayMethods = Object.create(arrayProto)

const methodsToPatch = ['push','pop','shift','unshift','splice','sort','reverse']
// 覆盖7个方法
methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method]
  def(arrayMethods, method, function mutator (...args) {
    // 执行原定任务
    const result = original.apply(this, args)
    // 通知
    const ob = this.__ob__
    // 如果操作是插入操作，还需要额外响应化处理
    let inserted
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observeArray(inserted)
    // notify change
    ob.dep.notify()
    return result
  })
})
```
下面看看defineReacive里做了什么

```js
//observer/index.js
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

* 创建一个消息订阅器 `Dep`,内部维护了一个数组，用来收集订阅者,下面分析
* 在`defineReactive`中，通过`Object.defineProperty`对数据进行劫持，将属性全部转为`getter/setter`
* 在获取值的时候会触发`get`函数，设置新值的时候回触发`set`函数，函数中执行相应的操作
* get函数中通过`dep.depend()`收集依赖
* set函数中通过触发dep.notify通知更新,调用订阅器的`update`方法

## 3.Dep

上面提到了一个新的东西 `Dep`,消息订阅器，内部有一个subs来管理依赖,看看代码是怎样实现的
```js
// onserver/dep.js
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;  //这里可以看出subs是用来存放Watcher的

  constructor () {
    this.id = uid++
    this.subs = []
  }

  //收集Watcher
  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)   //Dep.target就是Watcher
    }
  }
  //通知更新，调用Watcher的update方法
  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      //如果未运行异步，则不会在调度程序中对子进行排序
      //我们现在需要对其进行排序，以确保其正确触发
      //订单 
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
```
这里面主要是通过收集`Watcher`管理起来，上文说到，数据变化的时候调用set函数就会触发`notify`，来通知Watcher的update方法，下面看看Watcher是如何实现的

## 4.Watcher

`Watcher`(observer/watcher)的代码比较多，就挑一些主要代码讲解一下

在Watcher中的get函数
```js
get () {
    pushTarget(this)
    //...
}
```
而`pushTarget`函数是在observer/dep.js文件中
```js
export function pushTarget (target: ?Watcher) {
  targetStack.push(target)
  Dep.target = target
}
```
这里把Watcher赋值给了Dep.target,所以可以通过Dep.target直接拿到
在`defineReactive`的get函数中，直接使用`get`判断
还有上面dep文件下的，通过下列过程，让dep收集Watcher
```js
//defineReactive
if (Dep.target) {
  // 收集依赖
  dep.depend()
}
//src/core/observer/dep.js
depend () {
    if (Dep.target) {
      Dep.target.addDep(this)   //Dep.target就是Watcher
    }
  }
//src/core/observer/watcher.js
addDep (dep: Dep) {
  const id = dep.id
  if (!this.newDepIds.has(id)) {
    this.newDepIds.add(id)
    this.newDeps.push(dep)
    if (!this.depIds.has(id)) {
      dep.addSub(this)
    }
  }
}
//src/core/observer/dep.js
addSub (sub: Watcher) {
  this.subs.push(sub)
}
```
促使页面更新，更新函数update
```js
update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      /*同步则执行run直接渲染视图*/
      this.run()
    } else {
      /*异步推送到观察者队列中，下一个tick时调用。*/
      queueWatcher(this)
    }
  }
```
`queueWatcher`,更新队列，使用了异步更新方法，之后会有专门的文章讲解

看完Watcher的实现过程，就查找一下Watcher是在哪里调用的呢，上一篇文章说到了，在`mountComponent`函数中
```js
//src/core/instance/lifecycle.js
new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
```
在页面渲染的时候，就会创建一个Watcher，所以一个组件只有一个Watcher，同一组件的每个key值收集的Watcher都会是同一个，所有有key值变化，都会触发整个组件的Watcher更新，但是虚拟dom和diff算法(之后会讲)会帮我们精确计算，要更新的部分。

## 5.Compiler
 
现在整个数据响应式的流程还没有连通，初始化的时候，总得有事件会触发收集依赖吧，更新的时候也要触发重新渲染页面。
这些都发生在`compiler`，指令解析，渲染页面的时候，页面上有的值都会去触发get函数，如果data里的数据不管嵌套有多深，但是在页面上没有使用到，都是不会去收集的。
compiler的主要作用是对每个元素节点进行扫描和解析，根据指令模板替换数据，以及绑定相应的更新函数。
compiler里面代码比较复杂，之后再虚拟dom的时候一起讲解。

## 6.总结
![mvvm](https://www.eyes487.top/fs/uploads/1580382085171-data.png "图1")

* Observer： 数据监听器，能够监听属性的变化，实现数据响应式，首先要对数据进行劫持`(getter/setter)`，一个对象对应一个`observer`
* Watcher: 一个组件对应一个`Watcher`，每个`key`值会有一个订阅器`(Dep)`，如果key值在该组件中使用了，`Dep`就会收集当前Watcher，Watcher里面管理着更新方法`(update)`,当数据发生变化，就会触发update方法。Watcher是连接observer和compiler的桥梁。
* Compiler: 对每个元素节点进行扫描和解析，根据指令模板替换数据，以及绑定相应的更新函数。解析数据的时候就会触发`get`函数


-------------如果以上内容有不对的地方，还请大家指正------------


目录
[《Vue源码解析（一）：如何解读源码》](https://blog.eyes487.top/2020/01/26/vue-sourcec-analyse-01.html)
[《Vue源码解析（二）：new Vue() 初始化流程》](https://blog.eyes487.top/2020/01/27/vue-sourcec-analyse-02.html)
《Vue源码解析（三）：数据响应式》
[《Vue源码解析（四）：Vue批量异步更新策略》](https://blog.eyes487.top/2020/01/29/vue-sourcec-analyse-04.html)
[《Vue源码解析（五）：虚拟dom和diff算法》](https://blog.eyes487.top/2020/01/30/vue-sourcec-analyse-05.html)