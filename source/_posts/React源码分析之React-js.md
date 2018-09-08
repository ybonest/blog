---
title: React源码分析之React.js
date: 2018-09-08 18:50:43
tags: React源码系列
categories: React源码系列
---

&emsp;&emsp;使用React时，我们通常会`import React from 'react`,本章便用来列举React对象类所拥有的api以及源码，本系列React版本为`16.4.3-alpha.0`

<!--more-->

### Component与PureComponent

ES6语法中`Component`与`PureComponent`用来定义一个React组件，`Component`与`PureComponent`基本类似，但也有很大的不同，熟悉React的应该都知道，React组件会监听它的内部状态变化后，一旦状态变化后会重新触发渲染(render)，而监听变化自然会进行两次状态的比较，`Component`与`PureComponent`的不同便是在这层比较中发生的，`PureComponent`对组件的`props`与`state`进行的是浅比较，而`Component`则是深比较。

### Children

`Children`属性为组件的子组建提供了五个方法，分别是`map`，`forEach`、`count`、`toArray`、`only`

```javascript
React.Children.map(children, function[(thisArg)]);  // map遍历，与Array的map类似
React.Children.forEach(children, function[(thisArg)]);
React.Children.count(children); // 返回children中组件的总数
React.Children.only(children); // 判断是否唯一子集
React.Children.toArray(children) // 以数组形式重新返回
```

### Fragment

`React`组件中render返回元素是必须是单一元素（新版本中也可以以数组形式返回），但使用`Fragment`可以返回多元素。

```javascript
render() {
    return (
        <React.Fragment>
            this is first line.
            <h3>title</h3>
        </React.Fragment>
    )
}
```

### createRef



### React类源码
```javascript
import ReactVersion from 'shared/ReactVersion';
import {
  REACT_ASYNC_MODE_TYPE,
  REACT_FRAGMENT_TYPE,
  REACT_PROFILER_TYPE,
  REACT_STRICT_MODE_TYPE,
  REACT_PLACEHOLDER_TYPE,
} from 'shared/ReactSymbols';
import {enableSuspense} from 'shared/ReactFeatureFlags';

import {Component, PureComponent} from './ReactBaseClasses';
import {createRef} from './ReactCreateRef';
import {forEach, map, count, toArray, only} from './ReactChildren';
import {
  createElement,
  createFactory,
  cloneElement,
  isValidElement,
} from './ReactElement';
import {createContext} from './ReactContext';
import {lazy} from './ReactLazy';
import forwardRef from './forwardRef';
import {
  createElementWithValidation,
  createFactoryWithValidation,
  cloneElementWithValidation,
} from './ReactElementValidator';
import ReactSharedInternals from './ReactSharedInternals';

const React = {
  Children: {
    map,
    forEach,
    count,
    toArray,
    only,
  },

  createRef,
  Component,
  PureComponent,

  createContext,
  forwardRef,

  Fragment: REACT_FRAGMENT_TYPE,
  StrictMode: REACT_STRICT_MODE_TYPE,
  unstable_AsyncMode: REACT_ASYNC_MODE_TYPE,
  unstable_Profiler: REACT_PROFILER_TYPE,

  createElement: __DEV__ ? createElementWithValidation : createElement,
  cloneElement: __DEV__ ? cloneElementWithValidation : cloneElement,
  createFactory: __DEV__ ? createFactoryWithValidation : createFactory,
  isValidElement: isValidElement,

  version: ReactVersion,

  __SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED: ReactSharedInternals,
};

if (enableSuspense) {
  React.Placeholder = REACT_PLACEHOLDER_TYPE;
  React.lazy = lazy;
}

export default React;
```