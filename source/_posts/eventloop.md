---
title: 理解浏览器和Node中的事件循环(EventLoop)
categories: 前端
tag: ['Js','Node']
date: 2019-10-20
---


看过很多类似文章之后，想总结了一下关键点记录下来，方便之后回顾。
了解EventLoop可以用来分析一些异步次序的问题，同时还能了解一些浏览器和Node的内部机制。

# 前言: 为什么是事件循环

> 网景公司在1994年发明了第一个浏览器，之前网速都是非常慢的，所以等服务器反应可能需要很久，假如一个场景，表单填写，填写完成之后，等很久浏览器反应之后发现表单填写时错误的，这样多试几次之后，可能人都是崩溃的，所以网景这个公司花了7天的时间，设计了javascript，用来帮助用户可以在浏览器端做一些基础的校验，来减少交互的过程。可以看出，js是创建出来为了辅助网景公司销售他们的浏览器。
> 根据标准中对时间循环的定义描述，我们可以发现事件循环本质上是`user agent`(如浏览器端)用于协调用户交互（鼠标、键盘)、脚本（如js）、渲染（如HTML DOM、css样式）、网络等行为的一个机制。与其说是js提供了事件循环，不如说是嵌入js的`user agent` 需要事件循环来与多种事件源交互。



# 1.浏览器中的EventLoop

## 1.1 js引擎的事件循环机制

 * js是一门`单线程`、`无阻塞`的脚本语言。

 * 执行js文件的时候，会按照`从上到下`的顺序，把其中的同步代码加入`执行栈`中，然后按照`顺序`执行执行栈中的代码。

 * 执行`异步`代码的时候，他不会立即返回结果，会将这个异步事件挂着，当异步事件返回结果的时候，就会把这个事件加入`事件队列`

 * 他不会立即执行，直到执行栈中的所有任务都`执行完毕`，主线程处于`闲置`状态的时候，主线程就会去查询`事件队列`是都有任务

 * 有的话,就从中取出排在`第一位`的事件，把这个事件的`回调函数`加入执行栈，然后执行其中的代码

 * 如此反复，就形成了无线的循环，这个过程就叫做 `事件循环`


## 1.2 事件队列

事件队列也分为 **宏任务**（`macrotask`) 和 **微任务**（`microtask`）

![浏览器中的 EventLoop](https://www.eyes487.top/fs/uploads/1573386573224-eventloop1.png  "图1")

常见属于宏任务的有：
* `script(整体代码)`
* `setTimeout` / `setInterval`
* 事件回调
* 网络请求等，`XHR` 回调
* `history.back`

常见属于微任务的有：
* `Promise`
* `MutaionObserver`，是H5的新特性
* `Object.observe`(废弃)

在事件循环中，每进行一次循环操作称为tick
 * 在此次tick中最先进入的执行栈的任务(第一次是script代码)执行一次，执行完毕
 * 在执行栈为空的时候，主线程就会查看微任务列表是否为空
 * 存在微任务，就会不停地执行，直至微任务清空
 * 执行UI render
 * 下一次tick，执行任务队列排在第一的宏任务，执行完毕，可能会产生微任务
 * 是否有微任务，执行直至清空队列
 * 执行UI render
 * 主线程重复执行上述步骤

>浏览器完成一个宏任务，在下一个宏任务执行开始前，会对页面进行重新渲染。如果存在微任务，浏览器会清空微任务之后再重新渲染。

下面这张图可以说明流程
![浏览器中的 EventLoop](https://www.eyes487.top/fs/uploads/1580464442587-eventloop.jpg  "图2")




## 1.3 示例

```js
console.log(1);

setTimeout(() => {
  console.log(2);
  Promise.resolve().then(() => {
    console.log(3)
  });
});

new Promise((resolve, reject) => {
  console.log(4)
  resolve(5)
}).then((data) => {
  console.log(data);
  
  Promise.resolve().then(() => {
    console.log(6)
  }).then(() => {
    console.log(7)
    
    setTimeout(() => {
      console.log(8)
    }, 0);
  });
})

setTimeout(() => {
  console.log(9);
})

console.log(10);
```

正确的打印顺序是:
```js
1
4
10
5
6
7
2
3
9
8
```


分析，第一步：
* 先打印1 
* setTimeout加入事件队列（宏任务)
* 接着打印 4，promise的返回的回调加入事件队列（微任务）
* setTimeout加入事件队列（宏任务）
* 接着打印10

第二步：
* 微任务在宏任务之前执行，打印 5
* 此次微任务又产生了微任务，会在宏任务之前执行，接着打印 6
* 打印 7
* 把setTimeOut 加入事件队列(宏任务)

第三步：
* 此时，已经不存在微任务，按照顺序执行宏任务，打印2
* 执行时，又产生了一个微任务，只能先执行这个微任务，打印3
* 微任务执行完毕，按照顺序，执行宏任务，打印9
* 打印 8

> 所以，如果执行任务的时候不断的产生了微任务，那之后的宏任务就没办法执行了



# 2.Node中的EventLoop

Node的异步语法比浏览器更复杂，它可以和内核对话，所以它使用了`libuv`库来实现EventLoop，这个库负责各种回调函数的执行时间。

* **node中有一个注意的点**：
* 在`node v11`以下，它的执行顺序是先清空宏任务队列，然后在清空微任务队列，这样保证所有队列都有相等机会执行，这和浏览器执行顺序不一样
* 但是，在`node v11`开始，就修复了这个问题，执行顺序就和浏览器一致了，先执行一个宏任务，就清空微任务队列


## 2.1 Node中提供了四个定时器

* `setTimeout()`
* `setInterval()`
* `setImmediate()`
* `process.nextTick()`
上面两个语言标准，后两个是Node中独有的。
看看几个定时器在Node中的运行顺序:
```js
setTimeout(() => console.log(1));
setImmediate(() => console.log(2));
process.nextTick(() => console.log(3));
Promise.resolve().then(() => console.log(4));
(() => console.log(5))();

//node index.js
//5
//3
//4
//1
//2

```
> Node 规定，`process.nextTick`和`Promise`的回调函数，追加在本轮循环，即同步任务一旦执行完成，就开始执行它们。而`setTimeout`、`setInterval`、`setImmediate`的回调函数，追加在次轮循环,而本轮循环一定早于次轮循环执行。

所有同步任务执行完之后就是异步任务，process.nextTick是异步任务中最早执行的，如果想要异步任务尽快执行，就使用process.nextTick。
而promise的回调函数会被添加于微任务队列，追加在process.nextTick之后，也存在于本轮循环，只有等本轮循环(当前队列)执行完之后，才会进入下一队列。

## 2.2 事件循环的六个阶段

事件循环会按照顺序，反复地执行。每个阶段都有一个先进先出的回调函数队列。只有一个阶段的回调函数队列清空了，该执行的回调函数都执行了，事件循环才会进入下一个阶段。

![Node中的 EventLoop](https://www.eyes487.top/fs/uploads/1573386585444-eventloop2.jpg  "图3")

### 2.2.1 timers

这个是定时器阶段，处理`setTimeout()`和`setInterval()`的回调函数。进入这个阶段后，主线程会检查一下当前时间，是否满足定时器的条件。如果满足就执行回调函数，否则就离开这个阶段。

### 2.2.2 I/O callbacks

除了以下操作的回调函数，其他回调函数都在这个阶段执行
* setTimeout()和setInterval()的回调函数
* setImmediate()的回调函数
* 用于关闭请求的回调函数，比如socket.on('close', ...)

> 根据libuv的文档，一些应该在上轮循环poll阶段执行的callback，因为某些原因不能执行，就会被延迟到这一轮的循环的I/O callbacks阶段执行。换句话说这个阶段执行的callbacks是上轮残留的。

### 2.2.3 idle, prepare

该阶段只供 `libuv` 内部调用，这里可以忽略。

### 2.2.4 Poll

这个阶段是轮询时间，用于等待还未返回的 `I/O `事件，比如服务器的回应、用户移动鼠标等等。
这个阶段的时间会比较长。如果没有其他异步任务要处理（比如到期的定时器），会一直停留在这个阶段，等待 I/O 请求返回结果。

### 2.2.5 check

该阶段执行`setImmediate()`的回调函数。


### 2.2.6 close callbacks

该阶段执行关闭请求的回调函数，比如`socket.on('close', ...)`。

## 2.3 示例
来自官方文档的一个示例
```js
const fs = require('fs');

const timeoutScheduled = Date.now();

// 异步任务一：100ms 后执行的定时器
setTimeout(() => {
  const delay = Date.now() - timeoutScheduled;
  console.log(`${delay}ms`);
}, 100);

// 异步任务二：文件读取后，有一个 200ms 的回调函数
fs.readFile('test.js', () => {
  const startCallback = Date.now();
  while (Date.now() - startCallback < 200) {
    // 什么也不做
  }
});
```
分析：第一轮
* 没有到期的定时器
* 也没有刻意执行的 I/O 回调函数
* 内核读取文件
* Poll阶段，等待内核读取文件的结果，不会操作100ms，在定时器到期之前就会得到结果

第二轮
* 没有到期的定时器
* 有刻意执行的回调函数，进入`I/O callbacks`阶段，这个回调函数需要200ms，在执行到一半的时候，100ms定时器到期，但是必须等到这个回调函数执行完，才会离开这个阶段

第三轮
* 有了到期的定时器，所以会在timers阶段执行定时器，输出结果大概200多毫秒


## 2.4 setTimeout 和 setImmediate

由于setTimeout在 timers 阶段执行，而setImmediate在 check 阶段执行。所以，setTimeout会早于setImmediate完成。
```js
setTimeout(() => console.log(1));
setImmediate(() => console.log(2));

//结果不确定
//1  2  或者  2  1
```
因为
> setTimeout的第二个参数默认为0。但是实际上，Node 做不到0毫秒，最少也需要1毫秒，第二个参数的取值范围在1毫秒到2147483647毫秒之间。也就是说，setTimeout(f, 0)等同于setTimeout(f, 1)。

实际执行，进入事件循环，有可能到了1毫秒，有可能没有，这取决于系统当时的情况。如果没到1毫秒，那么 timers 阶段就会跳过，进入 check 阶段，先执行setImmediate的回调函数。

但是，下面代码一定是2   1
```js
const fs = require('fs');

fs.readFile('test.js', () => {
  setTimeout(() => console.log(1));
  setImmediate(() => console.log(2));
});
```
代码会先进入 I/O callbacks 阶段，然后是 check 阶段，最后才是 timers 阶段。因此，setImmediate才会早于setTimeout执行。


-------------如果以上内容有不对的地方，还请大家指正------------

Node中的事件循环参考 
[《Node 定时器详解》](http://www.ruanyifeng.com/blog/2018/02/node-event-loop.html)
[《不要混淆nodejs和浏览器中的event loop》](https://cnodejs.org/topic/5a9108d78d6e16e56bb80882)