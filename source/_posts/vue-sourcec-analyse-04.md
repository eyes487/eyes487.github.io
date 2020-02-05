---
title: Vue源码解析（四）：Vue批量异步更新策略
categories: 前端
tags: ['Vue','源码']
date: 2020-01-29
---


Vue页面中会存在多个组件，每个组件对应着一个Watcher，当页面中多个组件发生变化的时候，最好的办法就是把组件批量的更新，然后在一次性刷新页面，这样效率极高。这样想到了用`微任务`,会在同一次事件循环(tick)中执行完，然后在刷新页面。要了解这个知识点，首先需要了解一下`事件循环`，如果不了解可以先看看这篇文章[《理解浏览器和Node中的事件循环(EventLoop)》](https://blog.eyes487.top/2020/01/26/eventloop.html)

Vue版本: 2.6.11

## 1.实例
先看一个实例
```html
<div id="demo">
    <p id="num">{{count}}</p>
</div>
```
```js
const app = new Vue({
    el: '#demo',
    data:{
        count: 0,
    },
    mounted(){
        setTimeout(()=>{
            this.count++;
            console.log(num.innerHTML) //0
        },3000)
    }
})
```
可以看出count重新赋值后并不是马上刷新页面的，所以dom的更新是异步的。


## 2.Vue中的实现

下面是Vue官方文档中对异步更新队列的说明
> 可能你还没有注意到，Vue 在更新 DOM 时是异步执行的。只要侦听到数据变化，Vue 将开启一个队列，并缓冲在同一事件循环中发生的所有数据变更。如果同一个 watcher 被多次触发，只会被推入到队列中一次。这种在缓冲时去除重复数据对于避免不必要的计算和 DOM 操作是非常重要的。然后，在下一个的事件循环“tick”中，Vue 刷新队列并执行实际 (已去重的) 工作。Vue 在内部对异步队列尝试使用原生的 Promise.then、MutationObserver 和 setImmediate，如果执行环境不支持，则会采用 setTimeout(fn, 0) 代替。

* 异步： 只要侦听到数据变化，`Vue` 将开启一个队列，并缓冲在同一事件循环中发生的所有数据变更。
* 批量： 如果同一个 `watcher` 被多次触发，只会被推入到队列中一次。这种在缓冲时去除重复数据对于避免不必要的计算和 DOM 操作是非常重要的。然后，在下一个的事件循环“tick”中，Vue 刷新队列并执行实际 (已去重的) 工作。
* 异步策略: Vue 在内部对异步队列尝试使用原生的 `Promise.then`、`MutationObserver` 和 `setImmediate`，如果执行环境不支持，则会采用 `setTimeout(fn, 0)` 代替。

下面就通过源码来看一下，是如何实现的。


### 2.1 Watcher队列
数据变化触发更新是通过`dep.notify()`然后会去调用watcher的`update`方法
```js
//observer/watcher.js
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
其中有一些配置，比如`lazy`、`sync`，但是一般都不会设置，最主要的是`queueWatcher`，把左右Watcher添加进一个观察者对列,下面看看它的具体实现代码
```js
//observer/scheduler.js
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id

  // 去重，不存在才入队
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
     //如果已经刷新，则根据其ID拼接观察者
      //如果已经超过其ID，它将立即立即运行。 
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    //waiting变量，这是很重要的一个标志位，它保证flushSchedulerQueue回调只允许被置入callbacks一次。
    if (!waiting) {
      waiting = true

      if (process.env.NODE_ENV !== 'production' && !config.async) {
        flushSchedulerQueue()
        return
      }
      // 异步刷新队列
      nextTick(flushSchedulerQueue)
    }
  }
}

```
每个watcher都有一个id，要判断id是否存在于对列中，如果存在就不用入队了，有效地防止重复操作，然后执行`nextTick`,异步刷新对列，`flushSchedulerQueue`是一个函数，作用是循环执行`queue`中Watcher的run函数，用来更新视图,作为回调函数传入`nextTick`
看看它的具体实现
```js
//observer/scheduler.js
// nextTick的回调函数，在下一个tick时flush掉两个队列同时运行watchers
function flushSchedulerQueue () {
  currentFlushTimestamp = getNow()
  flushing = true
  let watcher, id
  /*
    给queue排序，这样做可以保证：
    1.组件更新的顺序是从父组件到子组件的顺序，因为父组件总是比子组件先创建。
    2.一个组件的user watchers比render watcher先运行，因为user watchers往往比render watcher更早创建
    3.如果一个组件在父组件watcher运行期间被销毁，它的watcher执行将被跳过。
  */
  queue.sort((a, b) => a.id - b.id)

  /*这里不用index = queue.length;index > 0; index--的方式写是因为不要将length进行缓存，因为在执行处理现有watcher对象期间，更多的watcher对象可能会被push进queue*/
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    /*将has的标记删除*/
    has[id] = null
    /*执行watcher*/
    watcher.run()
    // in dev build, check and stop circular updates.
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }

  // keep copies of post queues before resetting state
  /*得到队列的拷贝*/
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()
  /*重置调度者的状态*/
  resetSchedulerState()

  // call component updated and activated hooks
  /*使子组件状态都改编成active同时调用activated钩子*/
  callActivatedHooks(activatedQueue)
  /*调用updated钩子*/
  callUpdatedHooks(updatedQueue)

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush')
  }
}
```

### 2.2 nextTick

```js
//core/util/next-tick.js
export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  //把回调函数放入一个对列中
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  //pending是一个状态标记，保证timerFunc在下一个tick之前只执行一次
  if (!pending) {
    pending = true
    // 异步函数
    timerFunc()
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

这里主要是，把回调函数放入`callbacks`对列中，执行异步函数`timerFunc()`

### 2.3 timerFunc
看看timerFunc都做了哪些事情
```js
//core/util/next-tick.js
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    // 启动一个微任务
    p.then(flushCallbacks)
    //在有问题的UIWebViews中，Promise.then不会完全中断，但是
    //它可能会陷入怪异的状态，在这种状态下，回调被推入
    //微任务队列，但是直到浏览器才刷新队列
    //需要做一些其他工作，例如处理一个计时器。因此，我们可以
    //通过添加空计时器来“强制”刷新微任务队列。
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  //在本地Promise不可用的地方使用MutationObserver，
  //例如PhantomJS，iOS7，Android 4.4
  //（＃6466 MutationObserver在IE11中不可靠）
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  //回退到setImmediate。
  //从技术上讲，它利用了（宏）任务队列，
  //，但它仍然是比setTimeout更好的选择。 
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  // Fallback to setTimeout.
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}
```
在这个函数中，它会判断，看浏览器是否支持Promise，就启动一个微任务执行`flushCallbacks`，不支持就会做降级处理，主要是考虑到浏览器兼容性，按照`Promise`、`MutationObserver`、`setImmediate`、`setTimeout`这样的顺序，前两个是`微任务`，后两个是`宏任务`，`微任务`是首选，最后不得已要使用`宏任务`。我们知道`微任务`会在页面刷新之前执行完，使用`微任务`就可以比使用`宏任务`少执行一次`UI 渲染`

### 2.4 flushCallbacks
刷新的回调函数，这里才是真正执行刷新的地方
```js
//core/util/next-tick.js
function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}
```
会把之前`callbacks`队列中存放的所有回调函数全部取出来执行一遍，也就完成了整个刷新过程。

一直都知道，尽可能少的操作Dom，Vue批量异步更新也是如何，把所有要更新的数据都更新完了，在一次性刷新页面，这样效率是极高的，所以异步更新视图是极有必要的。


参考链接：
[https://github.com/answershuto...](https://github.com/answershuto/learnVue/blob/master/docs/Vue.js异步更新DOM策略及nextTick.MarkDown)
[https://segmentfault.com/a...](https://segmentfault.com/a/1190000013314893#item-6)


目录
[《Vue源码解析（一）：如何解读源码》](https://blog.eyes487.top/2020/01/26/vue-sourcec-analyse-01.html)
[《Vue源码解析（二）：new Vue() 初始化流程》](https://blog.eyes487.top/2020/01/26/vue-sourcec-analyse-02.html)
[《Vue源码解析（三）：数据响应式》](https://blog.eyes487.top/2020/01/26/vue-sourcec-analyse-03.html)
《Vue源码解析（四）：Vue批量异步更新策略》
[《Vue源码解析（五）：虚拟dom和diff算法》](https://blog.eyes487.top/2020/01/26/vue-sourcec-analyse-05.html)