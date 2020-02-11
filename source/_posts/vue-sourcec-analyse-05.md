---
title: Vue源码解析（五）：虚拟dom和diff算法
categories: 前端
tags: ['Vue','源码','虚拟DOM']
date: 2020-01-30
---


在Vue2中，数据响应式和虚拟DOM是分不开，它们是Vue的核心。Vue2中是一个组件一个Watcher实例，假如数据变化了，只能通知组件更新，这时就需要用到虚拟DOM和diff算法对比得出差异更新真实DOM了。

Vue版本: 2.6.11

## 1. 虚拟DOM
虚拟DOM(Virtual DOM)是对DOM的js抽象表示，他们是**js对象**，能描述**DOM结构和关系**，应用的各种状态变化会作用于虚拟DOM，最终映射到真实DOM上。因为是纯粹的js对象，所以操作起来就很高效。页面渲染的时候生成vdom，数据更新会生成一个新的vdom，**新的vdom**和**老的vdom**进行比较，这个过程称为**diff**算法，得出差异，应用在真实dom上。

Vue的diff算法是基于`snabbdom`算法所做的修改，感兴趣的朋友可以自己去查看。

下面从源码中去分析vue中的vdom是如何工作的。

## 2. 渲染和更新流程
首先，我们要找到，vm实例挂载的$mount会调用`mountComponent函数`(第二篇文章有说过)，看看这个函数
```js
//src/core/instance/lifecycle.js
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode
    // ...
  }
  callHook(vm, 'beforeMount')

  let updateComponent
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    updateComponent = () => {
    //   ...
    }
  } else {
      //用户$mount时，定义updateComponent
    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
  }
  //我们将其设置为观察者构造函数中的vm._watcher
  //因为观察者的初始补丁可能会调用$ forceUpdate（例如，在child内部
  //组件的挂接钩），它依赖于已定义的vm._watcher 
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
用户在`$mount`时，定义`updateComponent`函数
```js
updateComponent = () => {
    vm._update(vm._render(), hydrating)
}
```
下面 `new Watcher`把刚才定义的更新函数传进去了
```js
new Watcher(vm, updateComponent, noop, {
    before () {
        if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
        }
    }
}, true /* isRenderWatcher */)
```
如果`watcher`执行`run`方法，就会调用传入的回调，也就是`updateComponent`
而`updatemount`中需要执行`_render`方法,返回vdom，其中也就是给vdom额外的加了一些vue的属性，把返回的`vdom`作为`_update`的第一个参数，然后执行`_update`方法

那看看`_update`方法是如何工作的
```js
//src/core/instance/lifecycle.js
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    const vm: Component = this
    const prevEl = vm.$el
    const prevVnode = vm._vnode
    const restoreActiveInstance = setActiveInstance(vm)
    vm._vnode = vnode
    // Vue.prototype.__patch__ is injected in entry points
    // based on the rendering backend used.
    if (!prevVnode) {
      // 初始渲染 
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // 更新
      vm.$el = vm.__patch__(prevVnode, vnode)
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
    // updated hook is called by the scheduler to ensure that children are
    // updated in a parent's updated hook.
  }
```
在这里面，我们发现`vm.__patch__`方法，上面是初始化渲染，只要传入宿主元素和vdom就行，下面是更新，需要传入老的vdom和新的vdom。看看`patch`都做了什么,在`src/core/vdom/patch.js`
```js
return function patch (oldVnode, vnode, hydrating, removeOnly) {
    //判断是否有新的虚拟dom， 没有就删除
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    //是否有老的节点 1.初始化的时候，有传入$el, 2.更新，传入oldVnode
    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
        //是否是真正的dom
      const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // patch existing root node
        //更新的时候，真正的diff算法
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {
        //真实dom，就是初始化渲染
        if (isRealElement) {
          //...
          // 创建一个空节点并替换它
          oldVnode = emptyNodeAt(oldVnode)
        }

        // replacing existing element
        const oldElm = oldVnode.elm //当前宿主元素
        const parentElm = nodeOps.parentNode(oldElm) //即body，之后把新创建的真实dom追加进body，然后在删除之前的节点

        // create new node
        //根据虚拟dom创建真实dom
        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // update parent placeholder node element, recursively
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // 删除旧节点
        if (isDef(parentElm)) {
          removeVnodes([oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }

    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
```
首先会做一系列的判断，老节点和新节点的判断，更新操作,会执行`patchVnode`方法，初始化渲染，会根据虚拟dom生成真实dom追加在宿主元素的父节点中，然后在删除旧的节点。

下面看个例子
```html
<div id="demo">
    <p>虚拟dom</p>
    {{obj.foo}}
</div>
```
```js
const app = new Vue({
    el: '#demo',
    data:{
        obj:{
            foo:'hello world'
        }
    },
})
```
执行结果
![初始化渲染](http://fs.eyes487.top:9999/uploads/1580824895607-vdom.gif "图1")

## 3. diff算法

![同级比较](http://fs.eyes487.top:9999/uploads/1580889869180-compare.png "图2")

这是一张很经典的图，出自**《React’s diff algorithm》**，Vue的diff算法也同样，即仅在同级的vnode间做diff，递归地进行同级vnode的diff，最终实现整个DOM树的更新。那同级vnode diff的细节又是怎样的呢？

### 3.1 sameVnode

```js
//src\core\vdom\patch.js
function sameVnode (a, b) {
  return (
    a.key === b.key && ( // key值
      (
        a.tag === b.tag &&  // 标签名
        a.isComment === b.isComment &&  // 是否为注释节点
        isDef(a.data) === isDef(b.data) &&  // 是否都定义了data，data包含一些具体信息，例如onclick , style
        sameInputType(a, b)   // 当标签是<input>的时候，type必须相同
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```
在执行`patchVnode`之前会执行`sameVnode`判断，新老节点是否是否是同类型节点，值得进行`patchVnode`

### 3.2 patchVnode

先总结一下其中的规则，比较两个vnode，包括三种类型操作：**属性更新**，**文本更新**，**子节点更新**
具体规则如下：
* 新老节点均有children子节点，则对子节点进行diff操作，调用updateChildren
* 如果老节点没有子节点而新节点有子节点，先清空老节点的文本内容，然后为其新增子节点
* 当新节点没有子节点而老节点有子节点的时候，则先移除该节点的所有子节点
* 当新老节点都无子节点的时候，只是文本替换

那我们先看看`patchVnode`都做了什么
里面会有优化操作，比如`isAsyncPlaceholder`,`isStatic`,是否是占位符，静态节点，就不用了做`diff`了
```js
//src\core\vdom\patch.js
//判断是否是元素
if (isUndef(vnode.text)) { //没有文本就是元素
  //双方都有孩子
  if (isDef(oldCh) && isDef(ch)) {
    //比孩子，重排
    if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
  } else if (isDef(ch)) {
    //新节点存在
    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(ch)
    }
    //清空老节点文本
    if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
    // 创建孩子并追加
    addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
  } else if (isDef(oldCh)) {
    //老节点有孩子，直接删除即可
    removeVnodes(oldCh, 0, oldCh.length - 1)
  } else if (isDef(oldVnode.text)) { 
    //老节点存在文本，清空
    nodeOps.setTextContent(elm, '')
  }
} else if (oldVnode.text !== vnode.text) {
  //双方都是文本节点，更新
  nodeOps.setTextContent(elm, vnode.text)
}
```

这里面最主要的就是重排算法`updateChildren`，下面看看里面究竟是如何工作的

### 3.3 updateChildren

推荐[一篇文章](https://blog.csdn.net/m6i37jk/article/details/78140159)，用例子讲解的很详细，

比较两棵树最直接的方法就是双循环了，Vue中针对web场景做了特殊的**优化方式**。很多情况下我们都是在**前面**或者**后面**去追加节点，或者就是**升序**或者**降序**排列，那大概率的就是前面和后面的比较。有一个高效的方式，就是给不同节点设置`key`，就可以快速找到节点，进行判断是不是相同节点。

![vdom-diff1](http://fs.eyes487.top:9999/uploads/1580893492529-vdom-diff1.png "图3")

Vue中就是在新老两组vdom节点的左右两头设置了两对指针，在遍历的过程中，这几个指针都会向中间靠拢，当oldStartIdx > oldEndIdx或者newStartIdx > newEndIdx时结束循环。处理过的节点Vue会在oldVdom和newVdom中同时将它标记为已处理。

**下面看遍历规则**

**优先处理**的情况
* 头部的同类型节点、尾部的同类型节点, 这类节点更新前后位置没有发生变化，所以不用移动它们对应的DOM
* 头尾/尾头的同类型节点, 这类节点位置很明确，不需要再花心思查找，直接移动DOM就好

**原地复用**是Vue会尽可能的复用节点，不发生dom的移动。如果两个节点时同类节点(比如：div),那么Vue会直接复用DOM，这样的好处是不需要移动,但是也会产生一些问题。

**整个过程是逐步找到更新前后vdom的差异，然后将差异反应到DOM树上（也就是patch），特别要提一下Vue的patch是即时的，并不是打包所有修改最后一起操作DOM（React则是将更新放入队列后集中处理），朋友们会问这样做性能很差吧？实际上现代浏览器对这样的DOM操作做了优化，并无差别。**

先看看代码吧
```js
//src\core\vdom\patch.js
//循环条件：开始索引不能大于结束索引
while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
  if (isUndef(oldStartVnode)) {
    oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
  } else if (isUndef(oldEndVnode)) {
    oldEndVnode = oldCh[--oldEndIdx]
  } else if (sameVnode(oldStartVnode, newStartVnode)) {
    patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
    oldStartVnode = oldCh[++oldStartIdx]
    newStartVnode = newCh[++newStartIdx]
  } else if (sameVnode(oldEndVnode, newEndVnode)) {
    patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
    oldEndVnode = oldCh[--oldEndIdx]
    newEndVnode = newCh[--newEndIdx]
  } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
    //老的开始和新的结束节点相同，除了打补丁之外，向后移动
    patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
    canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
    oldStartVnode = oldCh[++oldStartIdx]
    newEndVnode = newCh[--newEndIdx]
  } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
    patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
    canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
    oldEndVnode = oldCh[--oldEndIdx]
    newStartVnode = newCh[++newStartIdx]
  } else {
    //4中猜想之后没有找到相同的，不得不开始循环查找
    if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
    idxInOld = isDef(newStartVnode.key)
      ? oldKeyToIdx[newStartVnode.key]
      : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
    if (isUndef(idxInOld)) { // New element
      //没找到则创建新元素
      createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
    } else {
      //找到除了打补丁，还要移动到对首
      vnodeToMove = oldCh[idxInOld]
      if (sameVnode(vnodeToMove, newStartVnode)) {
        patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        oldCh[idxInOld] = undefined
        canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
      } else {
        // same key but different element. treat as new element
        createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
      }
    }
    newStartVnode = newCh[++newStartIdx]
  }
}
//整理工作，必定有数组还剩下的元素未处理
if (oldStartIdx > oldEndIdx) {
  refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
  addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
} else if (newStartIdx > newEndIdx) {
  removeVnodes(oldCh, oldStartIdx, oldEndIdx)
}
```

首先，当 **oldStartVnode** 和 **newStartVnode** 或者 **oldEndVnode**和**newEndVnode** 满足同类节点，直接将该
VNode节点进行patchVnode即可，深层次比较，节点相同的两个指针向中间移动一个，不需再遍历就完成了一次循环。如下图，
![vdom-diff2](http://fs.eyes487.top:9999/uploads/1580895808862-vdom-diff2.png "图4")


如果**oldStartVnode**与**newEndVnode**满足同类节点。说明**oldStartVnode**已经跑到了**oldEndVnode**
后面去了，进行patchVnode的同时还需要将真实DOM节点移动到**oldEndVnode**的后面。这两个指针向中间移动一格。
![vdom-diff3](http://fs.eyes487.top:9999/uploads/1580896064990-vdom-diff3.png "图5")


如果**oldEndVnode**与**newStartVnode**满足同类节点，说明**oldEndVnode**跑到了**oldStartVnode**的前
面，进行patchVnode的同时要将**oldEndVnode**对应DOM移动到**oldStartVnode**对应DOM的前面。这两个指针向中间移动一格。
![vdom-diff4](http://fs.eyes487.top:9999/uploads/1580896064997-vdom-diff4.png "图6")


如果以上情况均不符合，则在old VNode中找与**newStartVnode**满足同类节点的vnodeToMove，如果找到了，就执行patchVnode，同时将vnodeToMove对应DOM移动到**oldStartVnode**对应的DOM的前面，**newStartVnode**向后移一格，但是在oldVnode中该节点处没有指针，所以就不能移动，只能该老节点标记一下说明它已经处理过了，设置为undefined
![vdom-diff5](http://fs.eyes487.top:9999/uploads/1580896065011-vdom-diff5.png "图7")


当然也有可能**newStartVnode**在old VNode节点中找不到一致的key，或者是即便key相同却不是同类节点，这个时候会调用createElm创建一个新的DOM节点。
![vdom-diff6](http://fs.eyes487.top:9999/uploads/1580896456382-vdom-diff6.png "图8")


至此循环结束，但是我们还需要处理剩下的节点。

当结束时oldStartIdx > oldEndIdx，这个时候旧的VNode节点已经遍历完了，但是新的节点还没有。说明了新的VNode节点实际上比老VNode节点多，需要将剩下的VNode对应的DOM插入到真实DOM中，此时调用addVnodes（批量调用createElm接口）。
![vdom-diff7](http://fs.eyes487.top:9999/uploads/1580896456391-vdom-diff7.png "图9")


但是，当结束时**newStartIdx** > **newEndIdx**时，说明新的VNode节点已经遍历完了，但是老的节点还有剩余，需要从文档中的节点删除。
![vdom-diff8](http://fs.eyes487.top:9999/uploads/1580896456394-vdom-diff8.png "图10")

至此，整个diff算法就结束了。


-------------如果以上内容有不对的地方，还请大家指正------------


参考链接：
[深入Vue2.x的虚拟DOM diff原理](https://blog.csdn.net/m6i37jk/article/details/78140159)
[Vue 虚拟DOM和Diff算法](https://blog.csdn.net/kameleon2013/article/details/89218685)


目录
[《Vue源码解析（一）：如何解读源码》](https://blog.eyes487.top/2020/01/26/vue-sourcec-analyse-01.html)
[《Vue源码解析（二）：new Vue() 初始化流程》](https://blog.eyes487.top/2020/01/27/vue-sourcec-analyse-02.html)
[《Vue源码解析（三）：数据响应式》](https://blog.eyes487.top/2020/01/28/vue-sourcec-analyse-03.html)
[《Vue源码解析（四）：Vue批量异步更新策略》](https://blog.eyes487.top/2020/01/29/vue-sourcec-analyse-04.html)
《Vue源码解析（五）：虚拟dom和diff算法》