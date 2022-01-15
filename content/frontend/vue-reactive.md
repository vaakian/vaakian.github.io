---
title: "Vue响应式原理"
date: 2022-01-14T21:22:48+08:00
draft: false
categories: ["Frontend","JavaScript"]
---

## 开始

众所周知，Vue的响应式实现方式有两种，在Vue2中通过「Object.defineProperty」重新定义`属性`的descriptor中的getter/setter实现来。而Vue3通过ES6的“新人”「Proxy」来实现。

## Vue2：Object.defineProperty

**网传**："Object.defineProperty无法监控到数组下标的变化，导致通过数组下标添加元素，不能实时响应；"

**纠错：**
**为什么Vue2中不劫持数组？**

- 数组的位置不固定，数量多变，正常对象key对应value一般不会变，但是如果数组删除了某个元素，比如第一个元素被删除或者头部增加一个元素，那么将导致后面所有的key对应value错位，如果6个元素，也就会触发5次set。

- 数组元素可能非常非常多，每个元素进行劫持有一定浪费，这可能是Evan you对性能的考虑。

- Vue2将数组的7个变异方法进行了重写，也就是更改了Array原型上的方法达到劫持变化。

- Vue2将对象新增属性/数组修改等操作，通过Vue.set来统一实现，这样其实是直接告诉Vue自己的set行为，即要新增/修改的属性名，然后重新劫持或者通知修改Deps[key].notify()，Vue2才可以监测到「新增数据/数组下标」的变化。


以下代码测试拦截数组会发生的情况
```javascript
const arr = [1, 2, 3, 4, 5, 6]

for (let key in arr) {
    let value = arr[key]
    Object.defineProperty(arr, key, {
        get() {
            console.log(`get: ${key}`)
            return value
        },
        set(newValue) {
            console.log(`set: ${key} to ${newValue}`)
            return value = newValue
        }
    })
}


arr[0] = 999 // 打印：set：0 to 999
arr[3] // 打印：get：3
arr.shift() // 会导致5次前移，所以产生5次get和5次set
/*
get: 1
set: 0 to 2
get: 2
set: 1 to 3
get: 3
set: 2 to 4
get: 4
set: 3 to 5
get: 5
set: 4 to 6
*/
```
## Vue3：Proxy
**网传：** Object.defineProperty只能劫持对象的属性，从而需要对每个对象，每个属性进行遍历，如果，属性值是对象，还需要深度遍历。Proxy可以劫持整个对象，并返回一个新的对象。

**纠错：** Proxy虽然是劫持的整个对象，但也是浅层劫持，属性值是对象时同样也需要深度遍历，不然该属性对象的变化也无法监测到。

```js
const obj = {
    count: 5,
    user: { name: 'JavaScript', age: 22 }
}


const proxiedObj = new Proxy(obj, {
    get(target, key) {
        console.log(`get：${key}`)
        return target[key]
    },
    set(target, key, value) {
        console.log(`set：${key} to ${value}`)
        return target[key] = value
    }
})


proxiedObj.user.name = 'Golang'
/*
只会触发一次get user即：
get：user
*/
```

怎么深度劫持？也很简单，一句递归解决，直接上代码。
```js
function deepProxy(obj) {
    return new Proxy(obj, {
        get(target, key) {
            console.log(`get：${key}`)
            if (typeof target[key] === 'object' && target[key] !== null) {
                return deepProxy(target[key]) // 递归劫持
              }
              return target[key]
          },
          set(target, key, value) {
              console.log(`set：${key} to ${value}`)
              return target[key] = value
          }
      })
  
  
  }
  const obj = {
      count: 5,
      user: { name: 'JavaScript', age: 22 }
  }
  
  
  const proxiedObj = deepProxy(obj)
  proxiedObj.user.name = 'Golang'/*
  控制台如下：
  get：user
  set：name to Golang
  */
```

所以为什么proxy优于Object.defineProperty？

从以上的例子就能看到，Object.defineProperty必须“预先”劫持属性。被劫持的属性才会被监听到。所以后添加的属性，需要手动再次劫持。

而proxy代理了整个对象，不需要预先劫持属性，而是在发表过微博/修改的时候，通过get/set方法来告诉你key，就算访问/修改对象上没有的属性，也会触发get/set。所以不管如何新增属性，总是能被捕获到。

测试访问和修改不存在的属性代码：
```js
  const obj = {
      count: 5,
      user: { name: 'JavaScript', age: 22 }
  }
    
  const proxiedObj = deepProxy(obj)
  // 访问和修改不存在的属性
  console.log(obj.someProp) // get: someProp
  obj.comment = 'JavaScript is Great!' // set: comment to JavaScript
  
```
⚠️不过由于Proxy是ES6的新特性，它在解决了一系列Vue2所存在的缺点的同时，也带来了兼容性的问题。

## THE END
学习不要停留于表面，耳听为虚，实践出真知。\
不要你抄我抄大家抄，最后反而错误结论流传千古。