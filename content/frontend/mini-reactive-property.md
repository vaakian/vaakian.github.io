---
title: "创建一个最简单的类Vue响应式数据"
date: 2022-03-25T13:02:55+08:00
draft: false
categories: ["Frontend", "Vue"]
---



## 依赖管理

每个属性通过Dep类管理依赖，当属性变更，则通过Dep.notify()通知依赖项更新。

```js
class Dep {
  subs = []
  addSub(sub) {
    this.subs.push(sub)
  }
  notify() {
    this.subs.forEach(sub => sub())
  }
  off() {
    this.subs = []
  }
}

module.exports = Dep
```


## 收集依赖

通过执行更新函数`watcherFn`，触发`getter`，此时收集到依赖函数`watcherFn`。依赖管理类Dep在`getter`中创建，如果没有依赖（即没有在watcherFn存在时触发getter），那就不会创建Dep对象，相对高效，节省资源。

### 处理computed属性

初始化时，执行`computed`函数，触发`getter`，收集相关依赖。
将每个`computed`的返回值也作为一个属性放到最终的代理属性上，并且与`data`同级别收集依赖。


```js
const Dep = require('./Dep')
// 暂存访问函数
let watcherFn

// 依赖收集过程：暂存 => 执行，访问属性，被收集 => 清除
const watcher = (fn) => {
  watcherFn = fn
  watcherFn()
  watcherFn = null
}

function Vue({ data, computed, watch }) {
  // 依赖管理类，每个属性key对应一个Dep对象
  const deps = {}
  const instance = new Proxy(data, {
    get(target, key) {
        // 如果watcherFn不存在，则不是在收集依赖（不是通过watcher函数触发的），直接返回该值即可
      if (watcherFn) {
        // 如果是watcherFn产生的get，才创建依赖。
        // Warn：只有watcherFn产生的get才有意义
        if (!deps[key]) {
          // 只有在访问时，如果没有dep，才添加依赖。
          // 可以节省一些空间
          deps[key] = new Dep()
        }
        deps[key].addSub(watcherFn)
      }
      return Reflect.get(target, key)
    },
    set(target, key, value) {
      const oldVal = target[key]
      const status = Reflect.set(target, key, value)
      // status是一个boolean，表示是否set成功
      if (status) {
        const dep = deps[key]
        // 如果是新属性，dep可能没有，只有被访问一次才会创建依赖管理
        dep?.notify()
        // 通知对应的watch函数，让函数的this始终指向instance
        watch[key]?.call(instance, value, oldVal)
      }
      return status
    }
  })

  for (const key in computed) {
    watcher(() => {
      // 让computed函数的this始终指向instance
      const result = computed[key].call(instance)
      // 将计算后的值同样的key放到实例中
      // 当该计算属性被set，那么也会触发相应的依赖更新，实现连锁更新
      Reflect.set(instance, key, result)
      // 如果设置data无法造成{连锁}反应更新
      // Reflect.set(data, key, result)
    })
  }
  return instance
}

```


## 测试

```js


const app = new Vue({
  data: {
    price: 5,
    discount: 0.8,
  },
  computed: {
    doublePrice() {
      return this.price * 2
    },
    discountedDoublePrice() {
      return this.discount * this.doublePrice
    }
  },
  watch: {
    price(newVal, oldVal) {
      console.log('price changed', newVal, oldVal)
    }
  }
})

console.log(app)

app.price = 80
console.log(app)

```

![](/images/2022-03-25-13-20-40.png)