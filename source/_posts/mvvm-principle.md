---
title: 实现MVVM原理
categories: 前端
tags: ['Vue','MVVM']
date: 2019-09-10
---

## 1.什么是MVVM

简单来说，`MVVM`就是一种模式，把**数据和视图进行关联**的一种模式。最常见的就是应用于`Vue`中，实现了数据绑定和试图渲染，Vue中主要通过`表单元素`设置`v-model`属性实现双向绑定。
放一张从网上找的图
![mvvm](http://fs.eyes487.top:9999/uploads/1573386600928-MVVM.png  "MVVM")

Vue中一般这样引用
```html
<div id="app">
    <input type="text" v-model="message">
    {{message}}
</div>
<script>
    let vm = new Vue({
        el: '#app',
        data:{
            message:'hello world'
        }
    })
</script>
```
了解Vue的朋友，一般都知道这些，这里对理论就不做过多阐述了，今天主要用代码实现MVVM原理。


## 2.Vue中的双向绑定

要实现数据双向绑定主要有三个核心点：
* 模板的编译
* 数据劫持,观察数据变化
* watcher
最后通过入口函数MVVM，整合三个核心点

首先自己创建一个MVVM.html
```html
<body>
    <div id="app">
        <input type="text" v-model="message.a"/>
        {{message.a}}
    </div>
    //这里会引用自己写的一些方法...
    <script src="watcher.js"></script>
    <script src="observer.js"></script>
    <script src="compile.js"></script>
    <script src="MVVM.js"></script>
    <script>
        let vm = new MVVM({
            el: '#app', //这里可以传字符串，也可以传document元素节点
            data:{
                message:  {
                    a: 'hello World!'
                }
            }
        })
    </script>
</body>
```
创建一个MVVM.js文件
```js
//MVVM.js
class MVVM{
    constructor(options){
        //把可用的东西挂载在实例上
        this.$el = options.el;
        this.$data = options.data;

        //如果有要编译的模板
        if(this.$el){
            //数据劫持，把对象的所有属性改为set和get方法
            new Observer(this.$data);
            this.proxyData(this.$data);
            //用数据和元素进行编译
            new Compile(this.$el,this)
        }
    }
    //把数据代理到this上，可以直接通过this取
    proxyData(data){
        Object.keys(data).forEach(key=>{
            Object.defineProperty(this,key,{
                get(){
                    return  data[key]
                },
                set(newValue){
                    data[key] = newValue
                }
            })
        })
    }
}
```
下面就开始用代码一步步实现MVVM原理。


### 2.1 模板编译
新建compile.js文件
```js
class Compile{
    constructor(el,vm){
        this.el = this.isElementNode(el)?el:document.querySelector(el);
        this.vm = vm;

        if(this.el){
            //开始编译
            //1.先把真是dom移入内存中Fragment（性能）
            let fragment = this.node2Fragment(this.el);
            //2.编译 =》提取想要的元素节点 v-model和文本节点{{}}
            this.compile(fragment)
            //把编译后的Fragment移回页面
            this.el.appendChild(fragment)
        }
    }

    /**
     * 
     * @param {辅助方法} node 
     */
    //判断是否是元素节点
    isElementNode(node){
        return node.nodeType === 1;
    }
    //是不是指令
    isDirective(name){
        return name.includes('v-')
    }

    /**
     * 核心方法
     * @param {*} el 
     */
    compileElement(node){
        //带v-model
        let attrs = node.attributes;//取出当前节点属性
        Array.from(attrs).forEach(attr=>{
            //判断属性名字是否包含v-
            let attrName = attr.name;
            if(this.isDirective(attrName)){
                //取到对应的值放到节点中
                let expr = attr.value;
                let [,type] = attrName.split('-');
       
                //node this.vm.$data 
                CompileUtil[type](node,this.vm,expr)
            }
        })
    }
    compileText(node){
        //带{{}}
        let expr = node.textContent; //取文本中的内容
        let reg = /\{\{([^}]+)\}\}/g;
        if(reg.test(expr)){
            //node this.vm.$data text
            CompileUtil['text'](node,this.vm,expr)
        }

    }

    compile(fragment){
        let childNodes = fragment.childNodes;
        Array.from(childNodes).forEach(node=>{
            if(this.isElementNode(node)){
                //是元素节点,编辑元素
                this.compileElement(node);
                //元素节点中可能还有节点
                this.compile(node)
            }else{
                //文本节点,编译节点
               this.compileText(node);
            }
        })
    }
    node2Fragment(el){
        let fragment = document.createDocumentFragment();
        let firstChild;
        while(firstChild = el.firstChild){
            fragment.appendChild(firstChild);
        }

        return fragment;
    }
}

CompileUtil ={
    getVal(vm,expr){//获取实例上对应的数据
        expr = expr.split('.');  //"message.a" => [message,a]
        return expr.reduce((prev,next)=>{ //vm.$data.a
            return prev[next]
        },vm.$data);
    },
    setVal(vm,expr,value){
        expr = expr.split('.');
        //收敛
        return expr.reduce((prev,next,currentIndex)=>{
            if(currentIndex === expr.length -1){
                return prev[next] = value;
            }
            return prev[next]
        },vm.$data)
    },
    getTextVal(vm,expr){
        return expr.replace(/\{\{([^}]+)\}\}/g,(...arguements)=>{
            return this.getVal(vm,arguements[1])
        })
    },
    text(node,vm,expr){ //文本处理
        let updateFn = this.updater['textUpdater'];
        expr.replace(/\{\{([^}]+)\}\}/g,(...arguements)=>{
            new Watcher(vm,arguements[1],newValue=>{
                //如果数据变化了，文本节点需要重新获取依赖的属性更新文本中的内容
                updateFn && updateFn(node,this.getTextVal(vm,expr))
            })
        })
        updateFn && updateFn(node,this.getTextVal(vm,expr))
    },
    model(node,vm,expr){ //输入框处理
        let updateFn = this.updater['modelUpdater'];
        //这里加一个监控，数据变化了。应该调用这个watch的cb
        new Watcher(vm,expr,newValue=>{
            updateFn && updateFn(node,this.getVal(vm,expr))
        })
        node.addEventListener('input',(e)=>{
            let newValue = e.target.value;
            this.setVal(vm,expr,newValue)
        })
        updateFn && updateFn(node,this.getVal(vm,expr))
    },
    updater:{
        //文本更新
        textUpdater(node,value){
            node.textContent = value;
        },
        //输入框更新
        modelUpdater(node,value){
            node.value = value;
        }
    }
}
```

### 2.2 数据劫持（Observer）

新建observer.js文件
```js
class Observer{
    constructor(data){
        this.observe(data)
    }
    
    observe(data){

        //要对这个data数据将原有的属性改成set和get的形式
        if(!data || typeof data !== "object"){
            return
        }
        //将所有数据一一劫持，先获取到data的key和value
        Object.keys(data).forEach(key=>{
            //劫持
            this.defineReactive(data,key,data[key]);
            this.observe(data[key]);//深度递归劫持
        })
    }
    //定义响应式
    defineReactive(obj,key,value){
        let that = this;
        let dep = new Dep();
        Object.defineProperty(obj,key,{
            enumerable: true, //可枚举
            configurable: true, //可修改
            get(){ //当取值的时候调用的方法
                Dep.target && dep.addSub(Dep.target)
                return value;
            },
            set(newValue){ //当给data属性中设置值的时候， 更改获取的属性的值
                if(newValue != value){
                    that.observe(newValue);
                    value = newValue;
                    dep.notify(); //通知所有人，数据更新了
                }
            }
        })
    }
}

class Dep{
    constructor(){
        //订阅的数组
        this.subs = [];
    }
    addSub(watcher){
        this.subs.push(watcher)
    }
    notify(){
        this.subs.forEach(watcher=>watcher.update())
    }
}
```

### 2.3 watcher
新建watcher.js文件,给需要变化的那个元素增加一个观察者，当数据变化后执行对应的方法
```js
class Watcher{
    constructor(vm,expr,cb){
        this.vm = vm;
        this.expr = expr;
        this.cb = cb;
        //先获取旧值
        this.value = this.get();
    }
    getVal(vm,expr){//获取实例上对应的数据
        expr = expr.split('.');  //"message.a" => [message,a]
        return expr.reduce((prev,next)=>{ //vm.$data.a
            return prev[next]
        },vm.$data);
    }
    get(){
        Dep.target = this;
        let value = this.getVal(this.vm,this.expr);
        Dep.target = null;
        return value;
    }
    update(){
        let newValue = this.getVal(this.vm,this.expr);
        let oldValue = this.value;

        if(newValue != oldValue){
            this.cb(newValue)
        }
    }
}
```

## 3. Proxy

在`Vue3.0`中将会使用Proxy实现数据的双向绑定，`Proxy` 是` ES6` 中新增的功能，它可以用来自定义对象中的操作。

```js
//语法
let p = new Proxy(target, handler);
//target    用Proxy包装的目标对象（可以是任何类型的对象，包括原生数组，函数，甚至另一个代理）。
//handlew   一个对象，其属性是当执行一个操作时定义代理的行为的函数。
```
数据劫持的简单实现
```js
function render() {
    console.log('模拟视图的更新')
}
let obj = {
    message: 'hello world',
}
let handler = {
    get(target, key) {
        // 如果取的值是对象就在对这个对象进行数据劫持
        if (typeof target[key] == 'object' && target[key] !== null) {
            return new Proxy(target[key], handler)
        }
        return Reflect.get(target, key)
    },
    set(target, key, value) {
        if (key === 'length') return true
        render();
        return Reflect.set(target, key, value)
    }
}


let proxy = new Proxy(obj, handler)
console.log(proxy.message) // hello world
proxy.message = 'my name is eyes487' // 支持新增属性
console.log(proxy.message) // 模拟视图的更新 my name is eyes487
```
用这种方法，代码可以精简很多，但目前proxy兼容性还不是很好。