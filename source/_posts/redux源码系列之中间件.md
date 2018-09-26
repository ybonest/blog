---
layout: blog
title: redux源码系列之中间件
date: 2018-09-15 17:48:05
tags: redux源码系列
categories: redux源码系列
---

redux工作的流程是`dispatch`一个`action`，触发`reducer`，改变`state`，并返回新的`state`状态，单纯的这个流程在处理一些异步请求、记录日志，以及增加项目的一些业务流程等这些副作用方面有些不便。在增加这些副作用说便需要用到`applyMiddleware`这个api，它的作用便是在`dispatch`与纯函数`reducer`之间增加了相应的函数，也就是所谓的`中间件`。

<!--more-->

### 中间件原理
我们都知道使用`createStore`创建`redux`后返回了`dispatch`函数，正常情况下`dispatch`调用后会直接触发`reducer`。而使用中间件后情况则大不一样，主观上，我们仍在使用`dispatch`，但实际上此`dispatch`已经原本的`dispatch`完全不同了。

首先我们来看一看`createStore`的源码

```js
import $$observable from 'symbol-observable'

import ActionTypes from './utils/actionTypes'
import isPlainObject from './utils/isPlainObject'

export default function createStore(reducer, preloadedState, enhancer) {
  if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
    enhancer = preloadedState
    preloadedState = undefined
  }

  if (typeof enhancer !== 'undefined') {
    if (typeof enhancer !== 'function') {
      throw new Error('Expected the enhancer to be a function.')
    }

    return enhancer(createStore)(reducer, preloadedState)
  }

  if (typeof reducer !== 'function') {
    throw new Error('Expected the reducer to be a function.')
  }

  let currentReducer = reducer
  let currentState = preloadedState
  let currentListeners = []
  let nextListeners = currentListeners
  let isDispatching = false

  function ensureCanMutateNextListeners() {
    if (nextListeners === currentListeners) {
      nextListeners = currentListeners.slice()
    }
  }

  function getState() {
    if (isDispatching) {
      throw new Error(
        'You may not call store.getState() while the reducer is executing. ' +
          'The reducer has already received the state as an argument. ' +
          'Pass it down from the top reducer instead of reading it from the store.'
      )
    }

    return currentState
  }

  function subscribe(listener) {
    if (typeof listener !== 'function') {
      throw new Error('Expected the listener to be a function.')
    }

    if (isDispatching) {
      throw new Error(
        'You may not call store.subscribe() while the reducer is executing. ' +
          'If you would like to be notified after the store has been updated, subscribe from a ' +
          'component and invoke store.getState() in the callback to access the latest state. ' +
          'See https://redux.js.org/api-reference/store#subscribe(listener) for more details.'
      )
    }

    let isSubscribed = true

    ensureCanMutateNextListeners()
    nextListeners.push(listener)

    return function unsubscribe() {
      if (!isSubscribed) {
        return
      }

      if (isDispatching) {
        throw new Error(
          'You may not unsubscribe from a store listener while the reducer is executing. ' +
            'See https://redux.js.org/api-reference/store#subscribe(listener) for more details.'
        )
      }

      isSubscribed = false

      ensureCanMutateNextListeners()
      const index = nextListeners.indexOf(listener)
      nextListeners.splice(index, 1)
    }
  }

  function dispatch(action) {
    if (!isPlainObject(action)) {
      throw new Error(
        'Actions must be plain objects. ' +
          'Use custom middleware for async actions.'
      )
    }

    if (typeof action.type === 'undefined') {
      throw new Error(
        'Actions may not have an undefined "type" property. ' +
          'Have you misspelled a constant?'
      )
    }

    if (isDispatching) {
      throw new Error('Reducers may not dispatch actions.')
    }

    try {
      isDispatching = true
      currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }

    const listeners = (currentListeners = nextListeners)
    for (let i = 0; i < listeners.length; i++) {
      const listener = listeners[i]
      listener()
    }

    return action
  }

  function replaceReducer(nextReducer) {
    if (typeof nextReducer !== 'function') {
      throw new Error('Expected the nextReducer to be a function.')
    }

    currentReducer = nextReducer
    dispatch({ type: ActionTypes.REPLACE })
  }

  function observable() {
    const outerSubscribe = subscribe
    return {
      subscribe(observer) {
        if (typeof observer !== 'object' || observer === null) {
          throw new TypeError('Expected the observer to be an object.')
        }

        function observeState() {
          if (observer.next) {
            observer.next(getState())
          }
        }

        observeState()
        const unsubscribe = outerSubscribe(observeState)
        return { unsubscribe }
      },

      [$$observable]() {
        return this
      }
    }
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

可以看到在没有中间件的情况下，`createStore`返回的是正常的对象api，而当传入中间件后则返回的是中间件被调用后的结果：
`return enhancer(createStore)(reducer, preloadedState)`;

> 下面来回顾一下中间件的使用:

redux提供了中间件处理函数`applyMiddleware`，可以将一系列的中间件进行整合。
通常我们这样创建store并传入中间件
```js
let store = createStore(
  todoApp,
  // applyMiddleware() 告诉 createStore() 如何处理中间件
  applyMiddleware(logger, crashReporter)
)
```

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
从源码上可以看出`applyMiddleware`是一个高阶函数，它可以接受一系列的中间件，并再返回一个高阶函数。

### 中间价中对原本api的相关代理
使用`applyMiddleware`混入中间件时总共为`redux`代理了两处，一处是`createStore`创建，一处是`dispatch`

> 对`createStore`的代理

回顾一下`createStore`传入中间件的情况，在`createStore`创建过程中会执行这样一句代码：
```js
return enhancer(createStore)(reducer, preloadedState)
```
可以看到在有中间件时，`createStore`执行到这里就被return了，此时开始执行`enhancer`。
这里面的`enhancer`就是调用`applyMiddleware`后返回的高阶函数，从源码中可以看出这个高阶函数接受`createStore`函数作为参数，然后返回一个接受`reducer`和`preloadedState`的函数（此处执行的是使用中间件时`createStore`的执行状态）。

最终，在`applyMiddleware`内部，不含中间件的`createStore`被重新执行，并将返回的状态保存在`applyMiddleware`函数内部的变量`store`上，这就是对`createStore`的代理实现。

> 对`dispatch`的代理

首先我们要明确一点，使用中间价时，`createStore`返回的时`applyMiddleware`执行后的结果，而`applyMiddleware`内部又重新执行了`createStore`,也就是上述所说的`createStore`代理问题，最终`applyMiddleware`又将原本该有的api重新返回，例如：`subscribe`,`getState`,`replaceReducer`，唯一发生变化的时`dispatch`.

从`applyMiddleware`源码中可以看到，`dispatch`被重新赋值了，而原本的`dispatch`被传入了中间件内部。
```js
const chain = middlewares.map(middleware => middleware(middlewareAPI))
dispatch = compose(...chain)(store.dispatch)
```

中间件是一个高阶函数，通常是这样定义的：
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



