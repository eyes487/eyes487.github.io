---
title: 手写DOM-Diff算法
categories: 前端
tags: ['React','虚拟DOM']
date: 2019-10-13
---
接触React已经两年多了,提起React，总是免不了要和Vue做一番对比，两个框架都使用了虚拟DOM。那今天就了解一下什么是虚拟DOM，虚拟DOM的实现和使用吧。

## 1.虚拟DOM

### 1.1 什么是虚拟DOM

 `Virtual DOM` 也就是虚拟节点。它通过js的Object对象模拟真实的DOM节点，然后在通过特定的 `render` 方法将其渲染成真实的DOM。在更改元素的时候并不是修改真正的DOM，而是通过虚拟DOM进行`Diff`运算，得到最小的差异生成补丁（`Patch`）应用到真实DOM中。

 ```bash
//createElemnt =>( type, props, children)

createElement('ul',{ class: 'list'},[
     createElemnt('li', {class: 'item'}, ['React']),
     createElemnt('li', {class: 'item'}, ['Vue']),
     createElemnt('li', {class: 'item'}, ['Angular'])
 ])
 ```
虚拟DOM就是通过上面这样一个方法，创建的一个js对象，如下：
![虚拟DOM](http://i.caigoubao.cc/626712/eyes487/v-dom.png  "图1")

### 1.2 创建虚拟DOM

可以通过`create-react-app`快速构建一个项目
下面我们就来实现这个`createElement`方法,创建一个element.js 和 index.js
```js
//element.js
class Element{
    constructor(type,props,children){
        this.type = type;
        this.props = props;
        this.children = children;
    }
}

function createElement(type,props,children){
    return new Element(type,props,children)
}

export {createElement}
```

```js
//index.js
import {createElement} from './element';

let virtualDom = createElement('ul',{class: 'list'},[
    createElement('li',{class: 'item'},['React']),
    createElement('li',{class: 'item'},['Vue']),
    createElement('li',{class: 'item'},['ANgular']),
])

console.log(virtualDom) //===>打印出虚拟DOM 如上图 图1
```


### 1.3 渲染真实DOM

通过 `render` 方法将虚拟DOM转化为真实DOM ,在通过`renderDOM`把真实DOM元素插入页面中
```js
//element.js    在element.js中添加如下方法
function setAttr(node,key,value){
    switch(key){
        case 'value':
            if(node.tagName.toUpperCase() === "INPUT" ||node.tagName.toUpperCase === "TEXTAREA"){
                node.value = value;
            }else{
                node.setAttribute(key,value);
            }
            break;
        case 'style': node.style.cssText = value;break;
        //可能还有其他情况...
        default: node.setAttribute(key,value); 
    }
}
//render方法可以将v-dom转化为真实dom
function render(eleObj){
    let el = document.createElement(eleObj.type);
    for(let key in eleObj.props){
        setAttr(el,key,eleObj.props[key]); //设置属性的方法
    }
    eleObj.children.forEach(child=>{  //遍历子节点，如果是虚拟dom继续渲染，否则代表是文本节点
        child = (child instanceof Element)?render(child) : document.createTextNode(child);
        el.appendChild(child);  //放入父节点
    })
    return el;
}
//把真实dom插入目标源
function renderDom(el,target){
    target.appendChild(el);
}
export {createElement,render,renderDom};
```

```js
//index.js
import {createElement,render,renderDom} from './element';

let virtualDom = createElement('ul',{class: 'list'},[
    createElement('li',{class: 'item'},['React']),
    createElement('li',{class: 'item'},['Vue']),
    createElement('li',{class: 'item'},['ANgular']),
])
let el = render(virtualDom)
console.log(virtualDom)
console.log(el);

renderDom(el,window.root)
```
这样界面上就生成了一个无序列表

## 2. DOM-Diff

### 2.1 DOM-Diff的作用

比较两个虚拟DOM的区别，根据两个虚拟DOM创建出补丁，描述改变的内容，将这个补丁用来更新DOM

### 2.2 DOM-Diff比较时遵循的原则

* 只会比较平级节点，不会跨级比较
* 比较平级节点时，如发现节点不存在，会将该节点及其子节点完全删掉，节点类型变了会直接生成新的节点
* 比较平级节点时，如果只是两个节点产生位置变化，那么会复用此节点，将两个节点交换位置即可，通过给同级列表元素添加key值实现
* DOM-Diff遵循 树的先序深度优先遍历

### 2.3 实现Diff算法

可以先定义一些规则: 
* 当节点类型相同，比较属性是否相同，如果不同，产生一个补丁包 {type:'ATTRs',attrs:{class: 'xxx'}}
* 当新dom节点不存在的时候，直接删除，补丁包 {type: 'REMOVE',index: xxx}
* 当节点类型不相同，就替换节点 {type: 'REPLACE',newNode: xxx}
* 当只是文本变化，就变更文本 {type: 'TEXT', text: 'xxxx'}
...

创建一个diff.js
```js
//diff.js
const ATTRS = 'ATTRS';
const TEXT = 'TEXT';
const REMOVE = 'REMOVE';
const REPLACE = 'REPLACE';
let Index = 0;
function diffAttr(oldAttrs,newAttrs){
    let patch = {};
    for(let key in oldAttrs){
        if(oldAttrs[key] !== newAttrs[key]){
            patch[key] = newAttrs[key]
        }
    }
    for(let key in newAttrs){
        if(!oldAttrs.hasOwnProperty(key)){
            patch[key] = newAttrs[key]
        }
    }
    return patch;
}
function diffChildren(oldChildren,newChildren,patches){
    oldChildren.forEach((child,idx)=>{
        recursion(child, newChildren[idx],++Index,patches)
    })
}
function isString(node){
    return typeof node === "string";
}
function recursion(oldNode,newNode,index,patches){
    let currentPatch = []; //每个元素都有一个补丁对象
    if(!newNode){
        currentPatch.push({type: REMOVE, index})
    }else if(isString(oldNode)&&isString(newNode)){
        if(oldNode !== newNode){
            currentPatch.push({type: TEXT, text: newNode})
        }
    }else if(oldNode.type === newNode.type){
        let attrs = diffAttr(oldNode.props,newNode.props)
        if(Object.keys(attrs).length>0){
            currentPatch.push({type: ATTRS,attrs})
        }
        //如果有子节点，遍历子节点
        diffChildren(oldNode.children,newNode.children,patches);
    }else{
        //说明节点被替换了
        currentPatch.push({type: REPLACE, newNode: newNode})
    }
    if(currentPatch.length>0){
        // 将元素和补丁对应起来，放入大的补丁包中
        patches[index] = currentPatch;
    }
}
function Diff(oldTree,newTree){
    let patches = {};
    //递归树 比较后的结果放入补丁包中
    recursion(oldTree,newTree,Index,patches);
    return  patches;
}


export default Diff;
```

```js
//index.js
import {createElement,render,renderDom} from './element';
import Diff from './diff';
let virtualDom1 = createElement('ul',{class: 'list'},[
    createElement('li',{class: 'item'},['React']),
    createElement('li',{class: 'item'},['Vue']),
    createElement('li',{class: 'item'},['Angular']),
])
let virtualDom2 = createElement('ul',{class: 'list2',id: 6},[
    createElement('li',{class: 'item'},['Html']),
    createElement('li',{class: 'item'},['Vue']),
    createElement('div',{class: 'item'},['Js']),
])

let patches = Diff(virtualDom1,virtualDom2)

console.log(patches) //===>打印出补丁包
```
补丁包如下 图2
![补丁包](http://i.caigoubao.cc/626712/eyes487/diff.png "图2")

### 2.4 把补丁包应用到真实DOM上

创建一个patch.js

```js
//patch.js
import { render ,Element,setAttr} from "./element";

let allPatches;
let index = 0;//默认打补丁的序号
function Patch(node,patches){
    allPatches = patches;
    
    walk(node)
    //给某个元素打补丁
}
function doPatch(node,patches){
    patches.forEach(patch=>{
        switch(patch.type){
            case 'ATTRS':
                for(let key in patch.attrs){
                    let value = patch.attrs[key];
                    if(value){
                        setAttr(node,key,value)
                    }else{
                        node.removeAttribute(key)
                    }
                }
                break;
            case 'TEXT': node.textContent = patch.text;
                break;
            case 'REPLACE': let newNode = (patch.newNode instanceof Element)?
                render(patch.newNode): document.createTextNode(patch.newNode);
                node.parentNode.replaceChild(newNode,node);
                break;
            case 'REMOVE': console.log('remove',node.parentNode);
            node.parentNode.removeChild(node);
                break;
        }
    })
}
function  walk(node){
    let currentPatch = allPatches[index++];
    let childNodes = node.childNodes;
    childNodes.forEach(child=>walk(child))
    if(currentPatch){
        doPatch(node,currentPatch)
    }
}
export default Patch;
```

```js
//index.js
import {createElement,render,renderDom} from './element';
import Diff from './diff';
import Patch from './patch';
let virtualDom1 = createElement('ul',{class: 'list'},[
    createElement('li',{class: 'item'},['React']),
    createElement('li',{class: 'item'},['Vue']),
    createElement('li',{class: 'item'},['Angular']),
])
let virtualDom2 = createElement('ul',{class: 'list2',id: 6},[
    createElement('li',{class: 'item'},['Html']),
    createElement('li',{class: 'item'},['Vue']),
    createElement('div',{class: 'item'},['Js']),
])

let el = render(virtualDom1)
renderDom(el,window.root)

let patches = Diff(virtualDom1,virtualDom2); //对比两个虚拟DOM的差异
Patch(el,patches); //把差异应用到元素节点

```
这样就完成了一个简易的DOM-Diff算法，当然还有很多情况没有考虑到，比如根据列表根据key值交换位置...