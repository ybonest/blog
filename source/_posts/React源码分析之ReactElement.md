---
title: React源码分析之ReactElement
date: 2018-09-11 22:30:48
tags: React源码系列
categories: React源码系列
---

&emsp;&emsp;`ReactElement`是`React`中的一些最基本接口，包括一个`React`对象的创建、拷贝与验证等。

<!--more-->

### 接口api

`ReactElement`提供了如下接口
* `createElement`
* `createFactory`
* `cloneElement`
* `isValidElement`

在介绍这些接口前需要先看一下`ReactElement`中一个最根本的方法`ReactElement`，它定义了是`React`最基本东西，便是`React`对象基本构成。

```js
const ReactElement = function(type, key, ref, self, source, owner, props) {
  const element = {
    // This tag allows us to uniquely identify this as a React Element
    $$typeof: REACT_ELEMENT_TYPE,

    // Built-in properties that belong on the element
    type: type,
    key: key,
    ref: ref,
    props: props,

    // Record the component responsible for creating this element.
    _owner: owner,
  };

  if (__DEV__) {
    // The validation flag is currently mutative. We put it on
    // an external backing store so that we can freeze the whole object.
    // This can be replaced with a WeakMap once they are implemented in
    // commonly used development environments.
    element._store = {};

    // To make comparing ReactElements easier for testing purposes, we make
    // the validation flag non-enumerable (where possible, which should
    // include every environment we run tests in), so the test framework
    // ignores it.
    Object.defineProperty(element._store, 'validated', {
      configurable: false,
      enumerable: false,
      writable: true,
      value: false,
    });
    // self and source are DEV only properties.
    Object.defineProperty(element, '_self', {
      configurable: false,
      enumerable: false,
      writable: false,
      value: self,
    });
    // Two elements created in two different places should be considered
    // equal for testing purposes and therefore we hide it from enumeration.
    Object.defineProperty(element, '_source', {
      configurable: false,
      enumerable: false,
      writable: false,
      value: source,
    });
    if (Object.freeze) {
      Object.freeze(element.props);
      Object.freeze(element);
    }
  }

  return element;
};
```

### createElement

`createElement`作用是创建一个`React`对象，在`React`中，无论以何种方式声明一个组件，最终都会转化成调用此函数创建一个`React`对象的形式，`createElement`接受三个参数，分别是
* type 组件类型
* config 组件所拥有的一些属性，比如`key`,`ref`等,`key`,`ref`最终会分别传给`ReactElement`中`React`对象所对应的部分，通常其余一些属性会被挂载到`React`对象的`props`(便是上面`ReactElement`定义的`props`)中
* children 顾名思义，第三个参数便是该React对象下的子元素，可以为其他`React对象`或者字符串等等类型。

```js
React.createElement('div', null, 'Hool Hool');

function ReactDemo({content, children}) {
  return (
    <div>
      {content}
      {children}
    </div>
    );
}
React.createElement(ReactDemo, {content: 'Hool Hool'}, 'Hool Hool Children')；
```

在使用ES6语法以及JSX语法时，我们通常这样定义一个组件

```js
import React from 'react';
export default class ReactDemo extends React.Component {
    render() {
        const { content, children } = this.props;
        return (
            <div>
                {content}
                {children}
            </div>
        );
    }
}
```
使用上面这个组件
```js
import React, { Component } from 'react';
import logo from './logo.svg';
import './App.css';
import ReactDemo from './example';
// 使用createElement创建React对象
const Demo = React.createElement(ReactDemo, {content: 'Hool Hool'},'Hool Children')

class App extends Component {
  render() {
    return (
      <div className="App">
        {/*使用Jsx语法*/}
        <ReactDemo content='Hool Jsx'>
          'Hool Jsx Children'
        </ReactDemo>
        {Demo}
      </div>
    );
  }
}
export default App;
```

上例中使用`createElement`与`Jsx`语法结果是一样的，就像上面所说的：使用`Jsx`最终也会被相应的`babel`转换为`createElement`创建形式


* `createElement`函数源码

```js
export function createElement(type, config, children) {
  let propName;

  // Reserved names are extracted
  const props = {};

  let key = null;
  let ref = null;
  let self = null;
  let source = null;

  if (config != null) {
    if (hasValidRef(config)) {
      ref = config.ref;
    }
    if (hasValidKey(config)) {
      key = '' + config.key;
    }

    self = config.__self === undefined ? null : config.__self;
    source = config.__source === undefined ? null : config.__source;
    // Remaining properties are added to a new props object
    for (propName in config) {
      if (
        hasOwnProperty.call(config, propName) &&
        !RESERVED_PROPS.hasOwnProperty(propName)
      ) {
        props[propName] = config[propName];
      }
    }
  }

  // Children can be more than one argument, and those are transferred onto
  // the newly allocated props object.
  const childrenLength = arguments.length - 2;
  if (childrenLength === 1) {
    props.children = children;
  } else if (childrenLength > 1) {
    const childArray = Array(childrenLength);
    for (let i = 0; i < childrenLength; i++) {
      childArray[i] = arguments[i + 2];
    }
    if (__DEV__) {
      if (Object.freeze) {
        Object.freeze(childArray);
      }
    }
    props.children = childArray;
  }

  // Resolve default props
  if (type && type.defaultProps) {
    const defaultProps = type.defaultProps;
    for (propName in defaultProps) {
      if (props[propName] === undefined) {
        props[propName] = defaultProps[propName];
      }
    }
  }
  if (__DEV__) {
    if (key || ref) {
      const displayName =
        typeof type === 'function'
          ? type.displayName || type.name || 'Unknown'
          : type;
      if (key) {
        defineKeyPropWarningGetter(props, displayName);
      }
      if (ref) {
        defineRefPropWarningGetter(props, displayName);
      }
    }
  }
  return ReactElement(
    type,
    key,
    ref,
    self,
    source,
    ReactCurrentOwner.current,
    props,
  );
}
```

### cloneElement
* 用法
```js
React.cloneElement(element, props, children);
```
`cloneElement`复制`React`组件，它的实现也十分简单，就是将要复制的组件以及props等相关属性系数拆分再合并形成新的`props`和`children`并再调用`cloneElement`，形成新的`React`对象

* `cloneElement`源码

```js
export function cloneElement(element, config, children) {
  invariant(
    !(element === null || element === undefined),
    'React.cloneElement(...): The argument must be a React element, but you passed %s.',
    element,
  );

  let propName;

  // Original props are copied
  const props = Object.assign({}, element.props);

  // Reserved names are extracted
  let key = element.key;
  let ref = element.ref;
  // Self is preserved since the owner is preserved.
  const self = element._self;
  // Source is preserved since cloneElement is unlikely to be targeted by a
  // transpiler, and the original source is probably a better indicator of the
  // true owner.
  const source = element._source;

  // Owner will be preserved, unless ref is overridden
  let owner = element._owner;

  if (config != null) {
    if (hasValidRef(config)) {
      // Silently steal the ref from the parent.
      ref = config.ref;
      owner = ReactCurrentOwner.current;
    }
    if (hasValidKey(config)) {
      key = '' + config.key;
    }

    // Remaining properties override existing props
    let defaultProps;
    if (element.type && element.type.defaultProps) {
      defaultProps = element.type.defaultProps;
    }
    for (propName in config) {
      if (
        hasOwnProperty.call(config, propName) &&
        !RESERVED_PROPS.hasOwnProperty(propName)
      ) {
        if (config[propName] === undefined && defaultProps !== undefined) {
          // Resolve default props
          props[propName] = defaultProps[propName];
        } else {
          props[propName] = config[propName];
        }
      }
    }
  }

  // Children can be more than one argument, and those are transferred onto
  // the newly allocated props object.
  const childrenLength = arguments.length - 2;
  if (childrenLength === 1) {
    props.children = children;
  } else if (childrenLength > 1) {
    const childArray = Array(childrenLength);
    for (let i = 0; i < childrenLength; i++) {
      childArray[i] = arguments[i + 2];
    }
    props.children = childArray;
  }

  return ReactElement(element.type, key, ref, self, source, owner, props);
}
```

### isValidElement
`isValidElement`校验是否是`React`对象，这个实现便更简单了，创建`React`对象时，会增加一个`$$typeof`属性，这个属性用来标记`React`对象

* 源码部分
```js
export function isValidElement(object) {
  return (
    typeof object === 'object' &&
    object !== null &&
    object.$$typeof === REACT_ELEMENT_TYPE
  );
}
```

其中`REACT_ELEMENT_TYPE`为
```js
const hasSymbol = typeof Symbol === 'function' && Symbol.for;
export const REACT_ELEMENT_TYPE = hasSymbol
  ? Symbol.for('react.element')
  : 0xeac7;
```