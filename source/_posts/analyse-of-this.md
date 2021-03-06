---
title: 浅谈this指向
categories: 前端
tag: Js
date: 2019-08-24
---
本文出自 《你不知道的js》，看过之后总是容易忘记，所以总结记录一下。

## 1.this到底是什么

>this是在 `运行时` 绑定的，并不是在编写时绑定，它的上下文取决于函数调用时的各种条件。this绑定和函数声明的位置没有任何关系，**只取决于函数的调用方式**。

>当一个函数调用时，**会创建一个活动记录（也称执行上下文）**。这个记录会包含函数在哪里调用（调用栈）、函数的调用方法、传入的参数等信息。this就是记录的其中一个属性，会在函数执行的过程中用到。


## 2.this绑定规则

### 2.1 默认绑定

非严格模式下，指向全局对象Windows

``` js
//严格模式
function foo(){
    "use strict";

    console.log(this.a);
}
var a = 2;
foo()// TypeError: this is undefined



function foo(){
    console.log(this.a);
}
var a = 2;
(function(){
    "use strict";

    foo()//2
})
```

### 2.2 隐式绑定


``` js
function foo(){
    console.log(this.a);
}
var obj ={
    a: 2,
    foo: foo
}
obj.foo(); //2
```

>无论是直接在obj中定义还是先定义在添加为引用类型，这个函数严格来说都不属于obj对象。然而，调用位置会使用obj上下文来引用函数，因此可以说函数被调用时obj对象拥有它或包含它。



``` js
//隐式丢失
function foo(){
    console.log(this.a);
}
var obj ={
    a: 2,
    foo: foo
}
var bar = obj.foo; // 函数别名
var a = "ooop"; //a是全局对象的属性
bar(); // "ooop"
```
>虽然bar是obj.foo的一个引用,但是实际上，它引用的是foo函数本身，所以此处应用了默认绑定。

参数传递也是一种隐式复制，传入函数时也会被隐式赋值

``` js
function foo(){
    console.log(this.a);
}
var obj ={
    a: 2,
    foo: foo
}
var a = "ooop"; //a是全局对象的属性

setTimeout(obj.foo, 100); //"ooop"
```

### 2.3 显式绑定

Js提供的绝大部分函数以及你自己所创建的函数都可以使用call()和apply()方法
``` js
function foo(){
    console.log(this.a);
}
var obj ={
    a: 2,
}
foo.call(obj); //2
```
这样还是会存在丢失绑定的问题

硬绑定（不会丢失绑定）
``` js
function foo(){
    console.log(this.a);
}
var obj ={
    a: 2,
}

var bar = function(){
    foo.call(obj);
}
bar(); //2
setTimeout(bar,100); //2

bar.call(Window); //2      硬绑定的bar不可能在修改它的this

//-----使用bind-----------
function foo(something){
    console.log(this.a, something);
    return thi.a + something;
}
var obj ={
    a: 2
}
var bar = foo.bind(obj);
var b = bar(3); // 2 3
console.log(b); //5
```
### 2.4 new 绑定

>首先我们重新定义一下JavaScript中的 “构造函数"。在js中，构造函数只是一些使用new操作符时被调用的函数。他们并不会属于某个类，也不会实例化一个类。实际上，他们甚至都不能说是一种特殊的函数类型，他们只是被new操作符调用的普通函数。

``` js
function foo(){
    this.a = a;
}
var bar = new foo(2);
console.log(bar.a); //2
```

## 3.优先级
* 函数是否在new中调用（new 绑定)?如果是的话this绑定的是新创建的对象。
  var bar = new foo();
* 函数是否通过call，apply（显示绑定）或者硬绑定调用？ 如果是的话，this绑定的是指定的对象
  var bar = foo.call(obj);
* 函数是否在某个上下文对象中调用（隐式绑定）？如果是的话，this绑定的是那个上下文对象。
  var bar = obj.foo();  
* 如果都不是的话，使用默认绑定。如果在严格模式下，就绑定到undefined，否则绑定到全局对象
  var bar = foo();

>就是这样。对于正常的函数调用来说，理解了这些知识点就可以明白this的绑定原理了。不过，凡事都有列外


## 4.绑定例外

### 4.1 
* 如果把null或者undefined做为this的绑定对象传入call、apply或者bind，这些值在调用时会被忽略。
* 一种非常常见的做法是使用apply（）来展开一个数组，并当做参数传入一个函数。类似地，bind（）可以对函数进行柯里化（预先设置一些参数）
``` js
function foo(a,b){
    console.log("a:" + a+ ",b:"+ b+);
}
//把数组展开成参数
foo.apply(null,[2,3]); //a:2, b:3

//使用bind（）进行柯里化
var bar = foo.bind(null,2);
bar(3); //a:2 , b:3
```
* 然而，总是使用null来忽略this绑定会产生一种副作用
* 更安全的做法是传入一个特殊对象，把this绑定到这个特殊对象不会对程序产生副作用
``` js
function foo(a,b){
    console.log("a:" + a+ ",b:"+ b+);
}
//我们的DMZ空对象
var の = Object.create(null);

//把数组展开成参数
foo.apply(null,[2,3]); //a:2, b:3

//使用bind（）进行柯里化
var bar = foo.bind(の,2);
bar(3); //a:2 , b:3
```

### 4.2 间接引用
调用这个函数会应用默认绑定规则
 ``` js
o.foo(); //3
(p.foo = o.foo)(); //2
```

### 4.3 软绑定 softBind()
把this绑定到指定对象上后，但是可以使用隐式绑定和显示绑定来更改this
```js
if(!Function.prototype.softBind){
    Function.prototype.softBind = function(obj){
        var fn = this
        var curried = [].slice.call(arguements, 1)
        var bound = function(){
            return  fn.apply(
                (!this || this === (window || global))?
                        obj : this
                    ,curried.concat.apply(curried, arguemnets)
            )
        }
        bound.prototype = Object.create(fn.prototype)
        return bound;
    }
}
```
```js
function foo(){
    console.log("name: "+this.name);
}

var obj1={name:"obj1"},
    obj2={name:"obj2"},
    obj3={name:"obj3"};

var fooOBJ=foo.softBind(obj1);
fooOBJ();//"name: obj1" 在这里软绑定生效了，成功修改了this的指向，将this绑定到了obj1上
 
obj2.foo=foo.softBind(obj1);
obj2.foo();//"name: obj2" 在这里软绑定的this指向成功被隐式绑定修改了，绑定到了obj2上
 
fooOBJ.call(obj3);//"name: obj3" 在这里软绑定的this指向成功被硬绑定修改了，绑定到了obj3上
 
setTimeout(obj2.foo,1000);//"name: obj1"
/*回调函数相当于一个隐式的传参，如果没有软绑定的话，这里将会应用默认绑定将this绑定到全局环境上，但有软绑定，这里this还是指向obj1*/

```

## 5.this词法
* ES6中有一种特殊函数类型：箭头函数
* 箭头函数会捕获调用函数时的this，一经绑定无法修改（new也不行）

>如果经常编写this风格的代码，但是绝大部分时候都会使用 self = this 或者 箭头函数 来否定this机制，那或许应该：
>* 只使用词法作用域并完全抛弃错误this风格的代码
>* 完全才用this风格，在必要时使用 `bind（）`，尽量避免使用 `self = this` 和 `箭头函数`