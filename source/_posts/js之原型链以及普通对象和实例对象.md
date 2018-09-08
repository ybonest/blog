---
layout: blog
title: js之原型链以及普通对象和实例对象
date: 2018-09-17 22:23:12
tags:
---

近日再看`redux`源码时，遇到一个方法利用原型链判断普通对象和实例对象的方式，记录一下其中原理。

<!--more-->

### 何为原型链
js中，每一个对象除去它本身所拥有的属性外，还拥有一个指针(__proto__)指向父类的属性，这样一级级指过去便形成了原型链。

### 普通对象
普通对象就是通过`Object`生成的对象，比如说`var a = {}`，a便是一个普通对象，`a.__proto__`指向了`Object`

### 实例对象
实例对象就是通过方法实例化形成的对象，比如说
```js
function abc(){}
var a = new abc();  // a就是方法实例化过后的对象
```
函数的原型链：创建一个函数，函数便会有一个`prototype`属性，该属性指向它的原型，原型中又会存在一个`constructor`属性，该属性又指向了函数本身，形成了一个闭环，除此之外，函数的原型也存在一个`__proto__`属性，该属性指向了`Object`，这是函数的原型链。

实例对象由于是在函数原型链基础上创建的对象，所以与普通对象的原型链有所不同，实例对象首先指向的是函数的原型，然后再由函数的原型指向`Object`;

### 如何判断普通对象与实例对象
判断普通对象与实例对象便是根据这两种对象的原型链的区别来判断的，首先普通对象的`__proto__`指向的是`Object`，而常规简单的实力对象通常需要`__proto__.__proto__`两级才能执行`Object`，所以便可根据此原理判断对象类型。

### 图解
![img](/images/prototype.png)

* redux源码中是这样判断的
```js
function isPlainObject(obj) {
    if (typeof obj !== 'object' || obj === null) return false;
    let proto = obj;
    while (Object.getPrototypeOf(proto) !== null) {
        proto = Object.getPrototypeOf(proto);  // 此处通过循环遍历最终取到原型链中点的Object
    }
    // 普通对象的第一级便是指向Object，而实例对象第一级指向它的实例化方法，
    // 所以此处若是普通对象，则返回true，实例对象返回false
    return Object.getPrototypeOf(obj) === proto;
}
```