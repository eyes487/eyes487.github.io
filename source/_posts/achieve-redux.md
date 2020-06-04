---
title: redux 简单实现
categories: 前端
tag: redux
date: 2020-02-26
---

> Redux 是 JavaScript 状态容器，提供可预测化的状态管理。Redux 除了和 React 一起用外，还支持其它界面库。 它体小精悍（只有2kB，包括依赖）。

## 1. redux用法

**三大原则**
* 单一数据源: 整个应用的 `state` 只存在于`唯一一个 store` 中
* State 是只读的: 唯一改变 state 的方法就是`触发 action`，action 是一个用于描述已发生事件的普通对象。
* 使用纯函数来执行修改: 描述 action 如何改变 state tree,都编写在`reducer`中

![redux工作流](http://fs.eyes487.top:9999/uploads/1583576247206-redux.jpg  "图1")

上面是redux工作流的图解。
`store`中存放数据，数据发生变化，通知页面更新。页面上`想要修改数据`，就会派发一些指令，通过`dispatch`发送`action` 给 `store`，`store`就会去查阅`Reducers`，在`Reducers`中定义了修改数据的方式，接收`之前的state`和`action`,返回`新的state`。


**下面回顾一下redux的用法:**
定义reducer函数
```js
const initState = {a: 1,b: 3}

function counterReducer(state = initState,action){
    const {type,payload} = action;
    switch(type){
        case "ADD":
            let a = state.a + payload;
            return {...state,a};
        case "MINUS":
            let b = state.b - payload;
            return {...state,b};
        default:
            return state;
    }
}
//通过createStore创建唯一数据源
const store = createStore(counterReducer);
```
在页面上通过dispatch,发送action更改数据
```js
store.dispatch({ type:'ADD', payload:5 });
```

通过subscribe订阅更新函数
```js
// 数据更新之后会执行订阅的回调函数
store.subscribe(()=>{
  //更新
});
```


## 2. redux实现
下面我们就来实现一个简单的redux

### 2.1 CreateStore
redux的几个核心函数， `getState`,`dispatch` ,`subscribe`

首先创建CreateStore函数
```js
export function createStore(reducer){
    //创建state，统一储存数据
    let state = undefined;
    //监听器数组
    let listenerMap = [];

    //getState():直接返回数据state
    function getState(){
        return state;
    }

    //dispatch: 执行action，会去reducer中查阅action执行，返回新的state
    //数据更新之后，监听器中存储的回调函数会被执行
    function dispatch(action){
        state = reducer(state,action)
        listenerMap.map(listener=>listener()) 
    }

    //subscribe: 订阅函数，把传入的回调函数，放入监听数组中
    function subscribe(listener){
        listenerMap.push(listener)
    }

    //初始化数据，会走到default，直接返回state
    dispatch({type: 'INIT'})

    return {
        getState,
        dispatch,
        subscribe
    }
}
```

* 首先，会定义`state`，用来存储所有的数据
* `listenerMap` 监听器数组，存放回调函数，方便之后数据更新之后执行回调
* `getState()` 直接返回数据state
* `dispatch` 会传入`action`，要执行的操作，定义的规则都放在`reducer`中，然后就会查阅`reducer`，执行具体方法，然后返回`state`，数据更新之后，执行监听器中的回调函数
* `subscribe` 订阅函数，把传入的回调函数，放入监听数组中
* `dispatch({type: 'INIT'})` 通过自己定义的规则，初始化数据，不直接通过外面给初始值，这样更严谨

### 2.2 combineReducers

有时候数据太多，需要拆分成多个reducer,所以redux中还提供了一个`combineReducers `函数，用来把Reducer 合成一个。
使用方法
```js
const reducers = combineReducers({reducerA, reducerB})
const store = createStore(reducers);
```
实现方法
```js
export function combineReducers(reducers){
    let combineReducers = {...reducers}; //复制一份新的

    let keys = Object.keys(combineReducers);
    return function combination(state = {},action){
        let hasChanged = false;
        const nextState = {};
        keys.forEach(key=>{
            const previousStateKey = state[key];
            const nextStateKey = combineReducers[key](previousStateKey, action)
            nextState[key] = nextStateKey;
            hasChanged = hasChanged || nextStateKey !== previousStateKey
        })
        return hasChanged? nextState : state
    }
}
```

* 复制一份新的传入的`reducers`，防止数据污染
* 聚合`reducers`之后，还是会返回一个函数，它接受的参数就和之前reducer一样
* 定义一个变量，记录`state`是否改变，初始值为`false`
* 循环这个对象中的`keys`值，获取到每个`reducer`对应的上一次的state值，保存到变量`previousStateKey`
* 通过当前`reducer`生成下一次的`state`，保存在`nextStateKey`
* 判断两次`state`的值是否相等，改变`hasChanged`变量
* 最后则是根据`hasChanged`来返回`state`，如果没有变化则返回原来的state。如果有变化则返回`nextState`。

### 2.3 applyMiddleware

官网描述
> 默认情况下，createStore() 所创建的 Redux store 没有使用 middleware，所以只支持 同步数据流。
你可以使用 applyMiddleware() 来增强 createStore()。虽然这不是必须的，但是它可以帮助你用简便的方式来描述异步的 action。

applyMiddleware用法
```js
//thunk:redux-thunk  用来做异步操作
const store = createStore(reducer,applyMiddleware(thunk,middlewareA,middlewareB));
```

在页面中就可以这样使用了
```js
asyncAdd =()=>{
    store.dispatch(dispatch=>{
        setTimeout(()=>{
            dispatch({type: 'ADD'})
        },1000)
    })
}
```
`store.dispatch`可以执行异步操作了，但是还是会返回dispatch，所以`applyMiddleware`的作用就是对所有中间件封装了一层，然后还是会返回store，store中会有dispatch

下面看具体实现
在createStore中
```js
export function createStore(reducer, enhancer){
    if(enhancer){
        return enhancer(createStore)(reducer)
    }
    //......
    return {
        getState,
        dispatch,
        subscribe
    }
}
```
`enhancer`,加强`createStore`函数，然后返回一个函数，接受reducer，还是和之前一样

下面看看`applyMiddleware`的具体实现方法
```js
export function applyMiddleware(...middlewares){
    //middlewares  中间件数组,接收中间件，然后会返回一个函数，相当于上文中的enhancer
    //enhaner会接收 createStore 参数， 返回一个函数
    //这个函数 ，接收reducer，执行原来createStore该做的操作，
    return createStore => (...args)=>{
        //创建store， 里面就有getState ，dispatch 等
        const store = createStore(...args);
        
        const middleApi = {
            getState: store.getState,
            dispatch: store.dispatch
        }
        //middlewares  传入的中间件数组，将middleApi传入，会返回一个新的函数数组
        const middlewareChain = middlewares.map(middleware=> 
            middleware(middleApi)
        )

        //通过把原始的dispatch，经过所有的中间件增强之后返回一个新的增加版dispatch，就可以做比如异步操作之类的了
        const dispatch = compose(...middlewareChain)(store.dispatch)
        return {
            ...store,
            dispatch
        }
    }

}
// 聚合函数  将上一个执行函数的返回值 ，传递给下一个参数
function compose(...funcs){
    return funcs.reduce((a,b)=>{
        return (...args)=>{
           return a(b(...args))
        }
    })
}
```


[源码地址](https://github.com/eyes487/react-source-learn)


-------------如果以上内容有不对的地方，还请大家指正------------