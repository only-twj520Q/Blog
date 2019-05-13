# redux源码浅读

## 前言

Redux是 JavaScript 状态容器，我们通常会在React的项目中搭配这个库用来处理复杂的业务逻辑。

在学习使用Redux一阶段以后，觉得有必要看一下源码，一方面提高对Redux的理解，另一方面也可以通过阅读源码来提高自身的程序设计水平。



## Redux基本使用

从实际使用出发，我们才能更加容易理解Redux的源码。

下面是Redux的基本使用。

```javascript
import { createStore } from 'redux';
const store = createStore(reducer, preloadedState, enhancer) 
// 设置监听函数
store.subscribe(listener);
// 使用
// 首先，用户派发action
store.dispatch(action);
// 然后，Store自动调用Reducer，并且传入两个参数：当前State和收到的Action。Reducer会返回新的State
let nextState = reducer(previousState, action);
// State一旦有变化，Store就会调用监听函数，监听函数更新state，重新render
function listerner() {
  let newState = store.getState();
  component.setState(newState);   
}
listener();

```

#### 流程图解

在利用Redux进行状态管理时，用户在UI层面触发行为，一个action对象通过`store.dispatch`派发到Reducer进行触发，接下来Reducer会根据type来更新对应的Store上的状态树，更改后的state会触发对应组件的重新渲染。

![](https://ws3.sinaimg.cn/large/006tNc79ly1g2w972tmz5j318a0o4gnp.jpg)

如果存在中间件的话，一个action对象在通过`store.dispatch`派发，在调用reducer函数前，会先经过一个中间件环节。在一个Redux架构中可以用多个中间件，这些中间件一起组织处理请求的“管道”。一个中间件是一个独立的函数，可以组合使用，中间件有一个统一的接口，正因为一个中间件只能完成一个特定的功能，所以把多个中间件组合在一起才能满足比较丰富的应用需求。当然在使用时，也需要按照顺序依次处理传入的action，只有排在前面的中间件完成任务之后，后面的中间件才能有机会继续处理action。

![](https://ws3.sinaimg.cn/large/006tNc79ly1g2w97n5xwdj31880fsdh3.jpg)



## 源码简介

先开看下 Redux 的源码的目录结构，没错，只有下面几个js，果然是短小精悍，只有2kb，而且生产环境几乎没有什么外部的依赖。虽然结构看起来并不复杂，内容也不是很长，但是Redux 作为函数式编程的典型代表，理解起来并不简单。

我们截取了 github上 的源码，如下图。

![image-20190510141319037](/Users/apple/Library/Application%20Support/typora-user-images/image-20190510141319037.png)

所有的js源码文件如下

- applyMiddleware.js     // Redux的插件机制实现
- bindActionCreators.js  // 将ActionCreator和dispatch绑定的工具
- combineReducers.js     // 将store分层的工具
- compose.js             // 将数个函数合并嵌套执行
- createStore.js         // 核心，生成store
- index.js               // 入口



## 源码解析

入口文件index.js和bindActionCreators这边就不做介绍了。下面展示的源码部分都是精简版的，去掉了一些注释和无关主逻辑的代码。

### createStore

首先来看createStore ，它的作用就是创建全局的store对象。在 Redux 中，整个应用只能有一个 Store，全局的state对象就保存在这个全局的store对象中。

#### 语法

* 参数
  * reducer
    * 数据类型：函数
    * 参数含义：接受当前state对象和action作为参数，用于处理state逻辑
  * preloadedState
    * 数据类型：：对象
    * 参数含义：初始的state
  * enhancer：
    * 数据类型：函数
    * 参数含义：store的增强器。这个函数由redux提供的applyMiddleware 函数来进行生成
* 返回值
  * store
    * 数据类型：对象
    * 该对象最常用的有三个属性getState，subscribe，dispatch。都是函数类型的

#### 源码

```javascript
import $$observable from 'symbol-observable'
import ActionTypes from './utils/actionTypes'
import isPlainObject from './utils/isPlainObject'

export default function createStore(reducer, preloadedState, enhancer) {
  // 判断 enhancer 是否是函数，enhancer 实际上就是中间件
  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error(...)
    }
    return enhancer(createStore)(reducer, preloadedState)
  }
  // 判断reducer是否是函数
  if (typeof reducer !== 'function') {
    throw new Error(...)
  }
  
	// 当前reducer
  let currentReducer = reducer
  // 当前状态state
  let currentState = preloadedState
  // 当前的监听器队列
  let currentListeners = []
  // 未来的监听器队列
  let nextListeners = currentListeners
  // 是否处于再派发action的状态，即是否在执行reducer函数。这是为了避免死循环。默认为false。
  let isDispatching = false
	// 这个函数用于确保currentListeners 和 nextListeners 是不同的引用
  function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
  }

  // 返回当前的state
  function getState() {
    if (isDispatching) {
      throw new Error(...)
    }
    return currentState
  }

  // 订阅监听函数
  function subscribe(listener) {
    if (typeof listener !== 'function') {
      throw new Error(...)
    }
    if (isDispatching) {
      throw new Error(...)
    }
    let isSubscribed = true
    // 这里Redux维护了两个队列，currentListeners和nextListeners
    // 考虑到只存在 currentListeners 的情况，如果我在执行某个 listener 中再次执行 subscribe或者 unsubscribe，会导致索引出错
    // 只有当下一次dispatch前，nextListeners又被同步给了currentListeners，之前的注册注销才会生效                
    ensureCanMutateNextListeners()
    nextListeners.push(listener)
    return function unsubscribe() {
      if (!isSubscribed) {
        return
      }
      if (isDispatching) {
        throw new Error(...)
      }
      isSubscribed = false
      ensureCanMutateNextListeners()
      const index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1)
    }
  }

	// 派发action
  function dispatch(action) {
    // action必须是一个对象
    if (!isPlainObject(action)) {
      throw new Error(...)
    }
    // type必须要有属性，不能是undefined
    if (typeof action.type === 'undefined') {
      throw new Error(...)
    }
    // 禁止在reducers中进行dispatch，因为这样做可能导致分发死循环
    if (isDispatching) {
      throw new Error(...)
    }
    try {
      isDispatching = true
      // 将当前的状态和action传给当前的reducer，用于生成最新的state 
      currentState = currentReducer(currentState, action)
    } finally {
      // 派发完毕
      isDispatching = false
    }	
    const listeners = (currentListeners = nextListeners)
    // 在得到新的状态后，依次调用每个监听器函数
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }
    return action
  }

  dispatch({ type: ActionTypes.INIT })

  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer,
    [$$observable]: observable
  }
}
```

#### 总结

createStore就是封装了一些api，通过闭包的形式，保存了两个重要的私有变量，state和listener。

**getState**：用来获取store中的state的。因为redux是不允许用户直接操作state，对于state的获取，是得通过getState的api来获取store内部的state。

**subscribe**：可以给 store 的状态添加订阅监听，一旦我们调用了 dispatch 来分发 action ，所有的监听函数就会执行。实际使用中用的不多，是因为react-redux 隐式的为我们帮我们完成了这方面的工作。

**dispatch**：是redux中一个非常核心的方法，也是我们在日常开发中最常用的方法之一。dispatch函数是用来触发状态改变的，它接受一个 action 对象作为参数，然后 reducer 就可以根据 action 的属性以及当前 store 的状态，来生成一个新的状态，从而改变 store 的状态；



### compose

compose 可以接受一组函数作为参数，从右到左来组合多个函数，然后返回一个组合函数。其实就是函数的合并，它想要实现的效果就像下面这样。

```javascript
compose(a, b, c) === (...args) => a(b(c(...args)))
```

#### 语法

- 参数
  - funcs：
    - 数据类型：函数
    - 参数含义：若干个用于组合的函数
- 返回值
  - func：
    - 数据类型：函数
    - 参数含义：执行该函数返回多个函数连续执行的结果

#### 源码

```javascript
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg
  }
  if (funcs.length === 1) {
    return funcs[0]
  }
  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```

underscore 的 compose 函数的实现，可以对比一下

```javascript
function compose() {
    var args = arguments;
    var start = args.length - 1;
    return function() {
        var i = start;
        var result = args[start].apply(this, arguments);
        while (i--) result = args[i].call(this, result);
        return result;
    };
};
```

#### 总结

compose函数理解并不困难，就是将多个函数依次进行组合，将后者的返回结果作为前者的参数。很多js库也都有类似的函数实现，redux利用了数组reduce方法的特性巧妙得用一行代码实现了功能。



###applyMiddleware

Redux 中最难的还是applyMiddleware的理解。这个插件机制是Redux一个很重要的功能，使得我们可以很方便的对action做各种各样的操作。比如日志打印，异步数据获取等等，只要添加一个middleware，就可以实现。这个类似于Express或者Koa中的中间件机制。

你可以这样理解函数柯里化，通过闭包保存了外部的一个变量，然后返回一个接收参数的函数，在该函数中使用了保存的变量，然后再返回值。

分析applyMiddleware的代码结构。 是个三级**柯里化**的高阶函数。它将依次获得三个参数：第一个是 middlewares 数组，第二个是 Redux 原生的 createStore，最后一个是 reducer。

#### 语法

- 参数
  - middlewares
    - 数据类型：函数
    - 参数含义：若干个中间件函数
- 返回值
  - func
    - 数据类型：函数
    - 参数含义：高阶函数，并作为函数creatStore的参数之一，连续执行该函数可以得到最终的store对象

#### 源码

```javascript
// 一个最简单的中间件
// 所有的中间件都必须遵循这样的格式，这个是因为applyMiddleware函数决定的
({getState, dispatch}) => next => action => next(action);

export default function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    // 利用传入的createStore和reducer创建一个store
    const store = createStore(...args)
    // 这个dispatch并不是最后在中间件里
    let dispatch = () => {
      throw new Error(...)
    }
    // 传给中间件第一层函数的参数                  
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    // 让每个 middleware 执行一遍，并传入两个参数getState和dispatch
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    // 上面提到compose的作用就是组合函数，结合中间件函数的格式，举个例子
    // 如果存在三个中间件a,b,c。在通过上面的函数执行过后，每个中间件返回的是next => action => {}这样的函数。函数的参数都是next函数，再返回一个函数，并通过闭包的形式把next函数保存起来。
    // 执行compose(...chain)(store.dispatch)
    // c最先执行，参数是store.dispatch，执行返回action=>{dosomethingC();return dispatch(action)}
    // b接着执行，参数是c的返回结果，然后返回action=>{dosomethingB();dosomethingA();return dispatch(action)}
    // a接着执行，参数是b的返回结果，然后返回action=>{dosomethingA();dosomethingB();dosomethingC();return dispatch(action)}，并且赋值给变量dispatch
    dispatch = compose(...chain)(store.dispatch)
    return {
      ...store,
      dispatch
    }
  }
}
```

#### 中间件的执行顺序

还有特别重要的就是关于中间件的执行顺序。下面是一个测试的demo。

```javascript
function middleware1({dispatch,getState}) {
  return function(next) {
    console.log('middleware1 next层',next);
    return function(action) {
      console.log('middleware1 action层 开始')
      next(action)
      console.log('middleware1 action层 结束')
    }
  }
}
function middleware2({dispatch,getState}) {
  return function(next) {
    console.log('middleware2 next层',next);
    return function(action) {
      console.log('middleware2 action层 开始')
      next(action)
      console.log('middleware2 action层 结束')
    }
  }
}
function middleware3({dispatch,getState}) {
  return function(next) {
    console.log('middleware3 next层',next);
    return function(action) {
      console.log('middleware3 action层 开始')
      next(action)
      console.log('middleware3 action层 结束')
    }
  }
}

let noop = ()=>{}
// 执行第一层函数
const chain = [middleware1, middleware2, middleware3].map(middleware => middleware({noop,noop}))
// 执行第二层函数
let dispatch = compose(...chain)(noop)
// 结果如下 
/**	
  middleware3 next层 ()=>{}
 	middleware2 next层 ƒ (action) {
    console.log('middleware3 action层 开始')
    next(action)
    console.log('middleware3 action层 结束')
   }
  middleware1 next层 ƒ (action) {
    console.log('middleware2 action层 开始')
    next(action)
    console.log('middleware2 action层 结束')
  }
**/

// 执行第三层函数
dispatch()
// 结果如下 
/**	
  middleware1 action层 开始
  middleware2 action层 开始
  middleware3 action层 开始
  middleware3 action层 结束
  middleware2 action层 结束
  middleware1 action层 结束
**/
```

#### 总结

中间件链的最内环是接受`store.dispatch`作为参数，最后返回的dispatch是增强的dispatch。每个中间件的next也是一次封装增强后的dispath。



### combineReducers

这个函数是用来整合多个reducers的， 因为createStore只接受一个reducer作为参数。

#### 语法

- 参数
  - reducer
    - 数据类型：对象
    - 参数含义：属性值是所有的reducer函数
- 返回值
  - finalreducer：
    - 数据类型：函数
    - 参数含义：reducer合并返回最终的reducer函数

#### 源码

```javascript
export default function combineReducers(reducers) {
  // 获取该对象的 key 值
  const reducerKeys = Object.keys(reducers)
  // 有效的reducer列表
  const finalReducers = {}
  // 过滤无效的reducer
  for (let i = 0; i < reducerKeys.length; i++) {
    const key = reducerKeys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (typeof reducers[key] === 'undefined') {
        warning(`No reducer provided for key "${key}"`)
      }
    }
    if (typeof reducers[key] === 'function') {
      finalReducers[key] = reducers[key]
    }
  }
  // 拿到过滤后的reducers的key值
  const finalReducerKeys = Object.keys(finalReducers)
	// 返回最终生成的reducer
  return function combination(state = {}, action) {
    // 定义state是否改变，默认false
    let hasChanged = false
    // 定义新的nextState
    const nextState = {}
    // 遍历reducers对象中的有效key，执行该key对应的value函数，即子reducer函数，并得到对应的state对象,将新的子state挂到新的nextState对象上
    for (let i = 0; i < finalReducerKeys.length; i++) {
      const key = finalReducerKeys[i]
      const reducer = finalReducers[key]
      // state树下的key是与 finalReducers下的key相同的
      const previousStateForKey = state[key]
      const nextStateForKey = reducer(previousStateForKey, action)
      nextState[key] = nextStateForKey
      // 如果存在触发某个reducer函数返回的state发生了改变，hasChanged置为true
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
    }
     // 遍历一遍看是否发生改变，发生改变了返回新的state，否则返回原先的state
    return hasChanged ? nextState : state
  }
}
```

#### 总结

这个combineReducers函数分为两部分，第一部分是检验传入的reducers的准确性，得到一个过滤后的对象 finalReducers。第二部分就是计算state，根据传入的action，遍历finalReducers，执行每个对象中的每个reducer函数，并将最终的state返回。注意的是，在reducer的代码处理逻辑中，如果没有返回一个新的state独享，而是在原有的state对象上增加或删除某个属性并返回。在比较新旧state的时候，会认为两个对象是一样的，最终不会



## 总结

Redux设计巧妙，思维严谨，整个代码没有特别臃肿的部分。

虽然个人还存在不理解的部分，比如creatstore的设计思想等等。

下面还要继续学习，下一步的阅读计划是react-redux和redux-thunk，希望能趁热打铁把react全家桶的相关代码都阅读学习一遍。

