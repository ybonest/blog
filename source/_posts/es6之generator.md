---
title: es6之generator
date: 2018-09-28 09:44:34
tags: ES6
categories: ES6
---

ES6中提供了`generator`语法，它是一种特殊类型的函数，可以返回多次，可以扁平化处理异步嵌套问题。

<!--more-->
### 基本语法
定义
```js
function* gen() {
  yield 1,
  yield 2
}
```
- `Generator.prototype.next()`:执行`generator`函数到下一个`yield`，next中可传递参数，赋值给yield前定义的变量
- `Generator.prototype.return()`: 返回结果，并结束`generator`执行状态，return可传递参数，作为最终返回结果，done状态变为true
- `Generator.prototype.throw()`: 抛出错误，并结束`generator`执行状态，throw可传递错误信息，并被try/catch捕获；

### 基本使用
- 函数定义方面`generator`函数与普通函数唯一的区别就是在function与函数名之间多了一个*，要注意的是`generator`不能用于箭头函数。
- `generator`函数调用时不会立即执行，它会返回一个`generator`对象，该对象调用`next`方法时开始执行，遇到`yield`停止执行，要想再次开始执行，则继续调用`next`方法
- `generator`函数可以有多个返回值，即`yield`和`return` 后面的值都会被返回。
```js
function* myGenerator() {
  console.log('hello generator');
  yield;
  console.log('middle generator');
  yield 'yield return';
  console.log('end generator');
  return 'end return';
}
const myGen = myGenerator(); 
myGen.next(); // 控制台打印hello generator，此处返回{value: undefined, done: false}
myGen.next(); // 控制台打印middle generator，此处返回{value: 'yield return', done: false}
myGen.next(); // 控制台打印end generator，此处返回{value: 'end return', done: true}
```
总的来说，`generator`函数中`yield`是一个分界线，每次执行时遇到`yield`就会停止，并以`{value: 'return value', done: bool}`对象形式返回`yield`后面的值。返回的对象`value`代表`yield`后的值，`done`表示当前执行状态，当再次`next`后，未遇到`yield`则为`true`;


### 处理异步

```js
function* myGen() {
  const a = yield time();
  return a;
}

function time() {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve(123);
    }, 2000);
  });
}

const myG = myGen();
/**
 *  第一次执行next，返回一个promise对象，在promise的then方法内，再次next，
 *  并把resove的值传入generator函数，这样a就获得了异步方法的结果。
 */
myG.next().value.then(data => {
  console.log(myG.next(data));
})
```

