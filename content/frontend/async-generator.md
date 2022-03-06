---
title: "implement async/await with generator/yield"
date: 2022-03-06T14:11:03+08:00
draft: false
categories: ["Frontend", "JavaScript"]
---

`async/await`不是魔法，它只是一个语法糖，底层与生成器generator密不可分。
这里以学习为目的，简单用一个例子来模拟它们的行为，能够有一个直观的理解。

需要前置知识，这里不再解释：
- [ES6 async/await](https://www.google.com/search?q=es6+async%2Fawait)
- [ES6 generator](https://www.google.com/search?q=es6+generator)

### 创建一个异步加法函数
延迟1秒返回加法结果，方便实验。
```js
function addAsync(x: number, y: number): Promise<number> {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve(x + y)
    }, 1000)
  })
}
```
### 实现单个yield（模拟await）
```js
function doAsync(g: () => Generator): void {
  // 调用异步函数main，返回一个可迭代的iterator
  const it = g()
  // 调用next，运行到第一个yield之后暂停
  const { value } = it.next()
  // 调用next则让yield之后的代码继续运行，并将v传递给yield作为返回值
  value.then(v => it.next(v))
}
function* main() {
  const result: number = yield addAsync(5, 2)
  console.log(result)
}
doAsync(main)
```
![bash](/images/2022-03-06-14-54-29.png)

### 多个yield
迭代支持多个yield，并处理异步/同步/字面量，和错误处理。
#### 异步函数执行器
简单的递归行为，结束一个promise之后，再允许生成器`g`继续执行，直到所有的`yield`执行完毕，即`done`为真时结束运行。
此处对value（为yield后面表达式的值）需要进行判断，可能是一个promise，也可能直接就是一个字面量值。
```js
function doAsync(g: () => Generator): void {
  const it = g()

  // 递归执行next，一个yield接一个。
  function iterate(lastValue?: any): void {
    const { value, done } = it.next(lastValue)

    // 停止迭代
    if (done) return

    // 方法一：如果value是Promise，则等待Promise完成，否则套一层静态Promise，则可以保证value是thenable的
    const promise = Promise.resolve(value)
    promise.then(iterate)

    // 方法二：手动判断
    // isThenable(value) ? value.then(iterate) : iterate(value)
  }

  iterate()
}

// function isThenable(obj: any) {
//   return obj && typeof obj.then === 'function'
// }
```


#### 运行

记住，await后面不一定是异步行为，可能是一个字面量，可能是一个同步任务。所以`doAsync`要做处理。

以下异步相当于 `async` 替换为 `*`，await替换为 `yield`。
```js
function* main() {
  const result1: number = yield 123
  console.log(result1)
  const result2: number = yield addAsync(5, 2)
  console.log(result2)
  const result3: number = yield addAsync(666, 21)
  console.log(result3)
}
doAsync(main)
```

结果：

![rest](/images/20220306-1.png)


#### 错误处理

如果promise是被reject捕获，则直接抛出一个错误。

怎么抛？

生成器函数(异步函数)返回值为it，那么就用`it.throw(e)`抛。

```js
const promise = Promise.resolve(value)
promise
  .then(v => iterate(v))
  .catch(err => it.throw(err))
```
剩下的太简单，懒得写了，就到这。


#### return处理

async函数的return结果，会被包装成一个Promise再返回出去。
那么需要做的就是：

- 返回一个 `new Promise((resolve, reject) => { })`
- 在`it.done`为`true`时，执行`reslove(it.value)`，结束运行
- ~~在执行代码过程中出现错误时 或 Promise被catch捕获，执行`reject(err)`或`it.throw(err)`，结束运行~~（待考证，先睡觉）
