---
layout: blog
title: redux源码系列之中间件
date: 2018-09-15 17:48:05
tags: redux源码系列
categories: redux源码系列
---

redux工作的流程是总的来说是`dispatch`推送`action`进入`reducer`纯函数，进而改变`state`，并返回新的`state`状态。中间件的作用就是在流程进入`reducer`之前添加一些副作用，比如说处理异步请求，日志记录等任务。

<!--more-->

### 中间件的使用
```js
import { applyMiddleware, createStore } from 'redux';
const logger = store => next => action => {
  console.log('is begin');
  next(action);
  console.log('is end');
}
const doSomething = store => next => action => {
  console.log('do something');
  next(action);
  console.log('do something end');
}
const reducer = function(state, action) {
  const newState = null;
  switch(action.type) {
    case 'DO_SOMETHING':
      newState = action.value;
      break;
    case 'DO_ENDING':
      newState = '';
      break;
    default:
      newState = state;
  }
  return newState;
}

const store = createStore(reducer, applyMiddleware(logger, doSomething));
```

### 中间件原理
```js
// 不使用中间件
const storeNoWare = createStore(reducer);
// 使用中间件
const storeHasWare = createStore(reducer, applyMiddleware(logger, doSomething));
```
此例中的`storeNoWare`与`storeHasWare`都是对象，这两个对象拥有相同的key。

|key|
| ---- |
|dispatch|
|subscribe|
|getState|
|replaceReducer|

这个几个属性中dispatch在中间件中被重新代理成了其他内容，而其余则完全不变。

下面是`applyMiddleware`的源码

```js
export default function applyMiddleware(...middlewares) {
  return createStore => (...args) => {
    const store = createStore(...args)
    let dispatch = () => {
      throw new Error(
        `Dispatching while constructing your middleware is not allowed. ` +
          `Other middleware would not be applied to this dispatch.`
      )
    }

    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    const chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  }
}
```
可以看出，`applyMiddleware`是一个高阶函数，它可以接受一系列的中间件，将各个中间件处理汇总，最后返回新的api。


### 中间价中对原本api的相关代理

`applyMiddleware`的主要作用是重新执行`createStore`创建，与`dispatch`的代理

> 对`createStore`的重新创建

回顾一下`createStore`传入中间件的情况，在`createStore`创建过程中会执行这样一句代码：
```js
return enhancer(createStore)(reducer, preloadedState)
```
可以看到在有中间件时，`createStore`函数在这里被返回，后续内容不再执行。而这里面的`enhancer`就是调用`applyMiddleware`后返回的高阶函数，从源码中可以看出这个高阶函数接受`createStore`函数作为参数，然后返回一个接受`reducer`和`preloadedState`的函数。
而最终，在`applyMiddleware`内部，`createStore`被重新执行，并返回无中间件时的执行结果。

> 对`dispatch`的代理

从`applyMiddleware`源码中可以看到，`dispatch`被重新赋值，而原本的`dispatch`被传入了中间件内部。
```js
const chain = middlewares.map(middleware => middleware(middlewareAPI))
dispatch = compose(...chain)(store.dispatch)
```

下面是一个中间件定义，它是个一个高阶函数。
```js
const logger = store => next => action => {
  console.log('dispatching', action)
  let result = next(action)
  console.log('next state', store.getState())
  return result
}
```
下面细细拆分一下中间件在`applyMiddleware`执行过程。
第一步，所有传入`applyMiddleware`的中间件均在下面代码中执行一次最外层的函数
```js
const chain = middlewares.map(middleware => middleware(middlewareAPI))
```
第二步，所有中间件外层函数执行后，返回的函数被存入了变量`chain`中，之后`chain`又经过了`compose`处理。
```js
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

最终所有函数会被处理成这样的形式返回：
```js
(...args) => a(b(c(d(...args))));
```

再看下面代码：
```js
dispatch = compose(...chain)(store.dispatch)
```
其实际执行的就是上述被`compose`处理返回后的函数，这样每一个函数执行的结果作为它前面一个函数的实参传到函数内部，而且`dispatch`是被最后一个函数接受的。本次执行后所有高阶中间件函数只剩了一层函数，这个函数接受最终的`action`。

最后一步，使用了中间件后，外界所得到的就是上步的最后一层函数，当使用`dispatch`时，`action`由外而内，最终传入接受真实`dispatch`的函数，触发`redux`状态改变。
