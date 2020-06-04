---
title: react-redux 简单实现
categories: 前端
tag: react-redux redux
date: 2020-03-07
---

要想在`react`中使用`redux`，通过`react-redux`会方便很多，今天就来看看react-redux究竟是如何实现的吧。


react-redux有两个核心关键模块: **Provider** 和 **connect**

## 1. react-redux用法

在根组件外面套一层`Provider`，使组件层级中都能获得`store`，store就是通过redux创建的。
```js
ReactDOM.render(
    <Provider store={store}>
        <App />
    </Provider>,
    document.getElementById('root')
);
```

在具体组件中通过`connect`获取`redux store`
```js
export default connect(
    //mapStateToProps
    (state) => {
        return ({count: state})
    },
    //mapDispatchToProps  可以是对象或者函数
    // {
    //     add: ()=>({type: 'ADD'}),
    // },
    dispatch=>{
        let res = {
            add: ()=>({type: 'ADD'}),
        }
        res = bindActionCreators(res,dispatch)
        return {
            dispatch,
            ...res
        }
    }
)(Component);
```
`connect`想当于一个高阶组件，接收一个组件，返回一个新的组件。
connect接受四个不同的参数，均为可选参数，但是一般最常用的是`mapStateToProps`和`mapDispatchToProps`,下面只看这两个参数的写法
通过上面这种方式，`redux store`就注入到组件中了，就可以在`this.props`中取到了


下面就看看`Provider` 和 `connect` 是怎么实现的吧


## 2. Provider实现
`Provider`向下面的子组件提供store，那下面的子组件如何跨层取得数据呢，这就要用到`Context`了，如果不了解context的朋友，请点击[这里](https://react.docschina.org/docs/context.html)

```js
//创建一个Context
const ValueContext = React.createContext();

export class Provider extends Component {
    render() {
        //通过Provider提供下去
        return <ValueContext.Provider value={this.props.store}>
            {this.props.children}
        </ValueContext.Provider>
    }
}
```
* 通过`React.createContext`创建一个Context
* 通过`ValueContext.Provider`提供给子组件
* 显示 `children`

## 3. connect实现

`connect` 会接收几个参数，主要实现`mapStateToProps`和`mapDispatchToProps`

创建`connect`函数，接收参数会返回一个新的函数，这个函数接收组件，返回新的组件
```js
export const connect = (mapStateToProps, mapDispatchToProps,) => WrapperComponent=> {
    return class extends Component{
        
        render(){
            return <WrapperComponent/>
        }
    }
}
```

在这函数里面，要想拿到父组件传递过来的`redux store`，可以通过`contextType`获取
```js
export const connect = (mapStateToProps, mapDispatchToProps,) => WrapperComponent=> {
    return class extends Component{
        static contextType = ValueContext; //上面provider处创建的context
        state= {

        }
        render(){
            console.log('this.context',this.context)//可以打印看看这时context已经有了
            return <WrapperComponent/>
        }
    }
}
```

下面实现`mapStateToProps`,在connect函数的class中
```js
componentDidMount(){
    const {getState} = this.context;

    let stateProps = mapStateToProps(getState());

    this.setState({
        props: {
            ...stateProps
        }
    })
}
```
* `this.context` 就是`redux store`，所以直接拿到`getState`，就可以获取`state`的值
* `mapStateToProps` 是一个会接收到`state`的参数，返回新的对象的一个函数
* `this.setState`可以让页面重新更新，所有使用它，来给`props`赋值，之后把这个`props`传入组件中

```js
 return <WrapperComponent  {...this.state.props}/>
```


下面实现`mapDispatchToProps`,在connect函数的class中,它或有两种形式，对象或者是函数
```js
const {dispatch} = this.context;
let dispatchProps;

if(typeof mapDispatchToProps === 'object'){
    dispatchProps = bindActionCreators(mapDispatchToProps,dispatch)
}else if(typeof mapDispatchToProps === 'function'){
    dispatchProps = mapDispatchToProps(dispatch)
}else{
    dispatchProps = {dispatch};
}

this.setState({
        props: {
            ...dispatchProps
        }
    })
```
* 从`this.context`中拿到`dispatch`
* 判断，如果传入的是对象，就给他包一层`dispatch`，就可以直接作为请求参数了,`bindActionCreators`是`redux`中提供的一个函数，下面 会写到它的实现原理
* 如果传入的函数，就一直作为函数返回，它会接收`dispatch`作为参数
* 默认，什么都不传的时候，就把`dispatch`返回回去
* 之后把这个`dispatchProps`参数放入`props`，就可以传递给组件了

bindActionCreators是redux中给提供的方法，只是函数包一层dispatch
```js
function bindActionCreators(actionCreators, dispatch) {
    var boundActionCreators = {};
  
    for (var key in actionCreators) {
      var actionCreator = actionCreators[key];
  
      if (typeof actionCreator === 'function') {
        boundActionCreators[key] = bindActionCreator(actionCreator, dispatch);
      }
    }
    return boundActionCreators;
}
function bindActionCreator(actionCreator, dispatch) {
    return function () {
      return dispatch(actionCreator.apply(this, arguments));
    };
  }
```
为了在使用`dispatch`改变数据之后，页面能够重新渲染，所以还需要使用`subscribe`订阅，用来监听数据改变，让页面发生变化。

好了，下面看看完整代码吧
```js
import React, { Component } from 'react'

const ValueContext = React.createContext();

export const connect = (mapStateToProps, mapDispatchToProps, mergeProps) => WrapperComponent=> {
    return class extends Component{
        static contextType = ValueContext;
        state ={
            props: {}
        }
        componentDidMount(){
            const {subscribe} = this.context;
            this.update()
            subscribe(this.update)
        }

        update =()=>{
            const {dispatch,getState} = this.context;
            let stateProps = mapStateToProps(getState());
            let dispatchProps ;

            if(typeof mapDispatchToProps === 'object'){
                dispatchProps = bindActionCreators(mapDispatchToProps,dispatch)
            }else if(typeof mapDispatchToProps === 'function'){
                dispatchProps = mapDispatchToProps(dispatch)
            }else{
                dispatchProps = {dispatch};
            }
            this.setState({
                props: {
                    ...dispatchProps,
                    ...stateProps,
                }
            })
        }
        render(){
            console.log('this.context',this.context);
            
            return <WrapperComponent {...this.state.props}/>
        }
    }
}



export class Provider extends Component {
    render() {
        return <ValueContext.Provider value={this.props.store}>
            {this.props.children}
        </ValueContext.Provider>
    }
}



function bindActionCreators(actionCreators, dispatch) {
    var boundActionCreators = {};
  
    for (var key in actionCreators) {
      var actionCreator = actionCreators[key];
  
      if (typeof actionCreator === 'function') {
        boundActionCreators[key] = bindActionCreator(actionCreator, dispatch);
      }
    }
    return boundActionCreators;
}
function bindActionCreator(actionCreator, dispatch) {
    return function () {
      return dispatch(actionCreator.apply(this, arguments));
    };
  }
```
这差不多就实现了一个简单的`react-redux`了。

[源码地址](https://github.com/eyes487/react-source-learn)



-------------如果以上内容有不对的地方，还请大家指正------------