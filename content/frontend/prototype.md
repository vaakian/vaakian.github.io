---
title: "Creating a Function and it's prototype"
date: 2022-04-19T15:48:02+08:00
draft: false
categories: ['JavaScript']
---

## 简单回顾 prototype

prototype 可以作为实例之间共享数据的一块空间。所以往 prototype 上放置任何属性，都可以被所有实例访问到。

```js
function Animal() {
  // this是实例本身
  console.log(this.name);
}
Animal.prototype.name = 'Animal';
Animal.prototype.greet = function () {
  console.log(`hello, i am ${this.name}`);
};
const cat = new Animal();
const dog = new Animal();
(cat.__proto__ === Animal.prototype) === dog.__proto__;
```

当通过实例本身访问任何属性，其过程为：`自身 => __proto__ => __proto__ 的 __proto__ => null`

所以

```js
const dog = new Animal();
// 相等
(dog.greet === Animal.prototype.greet) === dog.__proto__.greet;
```

但如果将构造函数改为

```js
function Animal() {
  this.greet = function () {
    console.log(`hello, i am ${this.name}`);
  };
}
Animal.prototype.greet = function () {
  console.log(`hello, i am ${this.name}`);
};
const dog = new Animal();
// 那么实例本身存在greet方法，则
dog.greet !== Animal.prototype.greet;
dog.greet !== dog.__proto__.greet;
```

## 那么函数的 prototype 是什么，与对象有什么区别？

最常用创建函数的方法，就是直接通过`function`关键字。但其实可以通过创建对象方式`new Function`来创建函数。
通过关键字`function`创建的函数，实为 JavaScript 的语法糖。

```ts
const getSix = new Function('return 6');
console.log(getSix);
// 最后一个参数为函数的代码逻辑，前面的为函数参数名
const add = new Function('a', 'b', 'return a + b');
console.log(add(1, 2)); // 3
console.log(add.length); // 2
```

所以对于 JavaScript 来说，函数也是一个对象(object)，拥有对象的所有特征。\
所以下面代码也就容易理解了。

```ts
const getSix = new Function('return 6');
console.log(getSix.__proto__ === Function.prototype); // true
```
