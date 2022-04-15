---
title: "React18: useDeferredValue"
date: 2022-04-13T12:24:53+08:00
draft: false
categories: ["Frontend", "React"]
---

## `useDeferredValue`的功能 & 场景

想象一个非常耗时的组件`SlowComponent`，当每次重新渲染时，在渲染结束之前，整个`JavaScript`主线程被阻塞(block)，无法处理任何事件，处于假死状态。

所以以下的`input`在输入新的值时，需要等到所有组件被`render`，`input`框才会更新显示新的值。\
而`SlowComponent`耗时大约`1s`才返回结果，所以从键盘输入 => 到页面上显示结果，需要等待`1s`。\
包括`input`输入框。

```js
function App() {
  const [text, setText] = useState("hello")
  return (
    <div className="App">
      <input value={text} onChange={e => setText(e.target.value)} />
      {/* SlowComponent组件非常耗时 */}
      <SlowComponent text={text} />
    </div>
  )
}
// 通过React.memo优化，text不变时，返回缓存的渲染结果。
const SlowComponent = React.memo(({ text }) => {
  const now = Date.now()
  while (Date.now() - now < 500) {
    // do nothing
  }
  return <div>{text}</div>
})
```

在线测试，改变1次输入框，输入框需要在`500ms`后响应。\
https://codesandbox.io/s/goofy-khayyam-3q410q?file=/src/App.js

那么可不可以先立刻渲染出`input`，再去渲染`SlowComponent`呢？\
答案是可以的，将「渲染过程」从上面的1次，分成2次。

怎么分？就是`useDeferredValue`可以用到的场景：

> 它可以让一个state在单次渲染中，先返回上一次的值，当渲染任务结束时，再更新最新值，然后再触发一次渲染。

所以得到下面修改后的代码，`input`框的响应非常及时，不会被`SlowComponent`阻塞。

```js
import { useDeferredValue } from 'react'

function App() {
  const [text, setText] = useState("hello")
  const deferredText = useDeferredValue(text)
  console.log({text, deferredText})
  return (
    <div className="App">
      <input value={text} onChange={e => setText(e.target.value)} />
      {/* SlowComponent组件非常耗时，当text更改导致re-render时
      deferredText会在第一次渲染时保持上次的值（即不变），
      所以这个组件在第一次是返回缓存的结果，不会非常耗时。
      当第一次渲染结束后，
      再更新最新的deferredText值，从而再触发一次渲染，
      此时才会重新渲染SlowComponent */}
      <SlowComponent text={deferredText} />
    </div>
  )
}
```

控制台结果
![](/images/20220412215221.png)

在线测试，改变**1**次输入框的内容，输入框能过及时得到响应。\
https://codesandbox.io/s/awesome-dewdney-rb9epf?file=/src/App.js

## 手动实现

知道了`useDeferredValue`的功能，那么这里也能过手动模拟一下它的行为，经测试与原版`useDeferredValue`效果一致。

```js
function useDeferredValue2(newVal) {
  const [current, setCurrent] = useState(newVal)
  useEffect(() => {
    // 如果值变了，先执行return current（就是上一次的值）
    // 渲染结束后，再更新最新的值，然后再触发一次更新
    setCurrent(newVal)
  }, [newVal])
  return current
}
```

## 小结

到这里，就明白了`react18`的重心是放到了「异步（并发）渲染（concurrent rendering）」上，让更加紧急的渲染先完成，最后再去渲染相对来说不那么紧急的任务。或者是说`短任务优先`，让耗时小的任务先做，耗时长的任务后做。从而提高页面的响应性（more responsive）。

### 本文的在线测试链接

未优化，输入框更新响应慢。\
https://codesandbox.io/s/goofy-khayyam-3q410q?file=/src/App.js

优化后，输入框更新响应及时。\
https://codesandbox.io/s/awesome-dewdney-rb9epf?file=/src/App.js

### 剩下的问题

优化后，输入框只改变1次时，能够及时响应。但响应完之后渲染`SlowComponent`的期间，再改变输入框内容，又会被阻塞。
解决办法可以对`setText`事件做一个防抖，在设定的阈值时间内触发的更改，永远只更新最后一次。