---
title: "React18 updates"
date: 2022-04-12T14:56:18+08:00
draft: false
categories: ["Frontend", "React"]
---

## `React18`更新了什么？

最大的更新就是`并发渲染(concurrent rendering)`特性了。这里将同步渲染和并发渲染进行对比，以及了解和测试它所影响的视觉行为。

其次，使用`react18`，渲染方式有小的变动如下：

before
```js
import React from 'react'
import ReactDOM from 'react-dom'
import './index.css'
import App from './App'

ReactDOM.render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
  document.getElementById('root')
)
```

after
```js
import { React } from 'react'
import { createRoot } from 'react-dom/client'
import './index.css'
import App from './App'

const root = createRoot(document.getElementById('root'))
root.render(<App />)
```
### 并发渲染 vs 同步渲染
在同步渲染过程中，每个组件所访问的外部资源都能够保证一致，因为`render`是`一个大的任务`。在这个任务完成之前，所有的任务都会被阻塞，包括页面。如果进行一个很重的渲染工作，在完成渲染之前，页面就会卡住无响应。

而并发渲染则不同，一个`render`可以被拆分成多个片段，在这个期间可以页面可以继续响应，不会卡死。但同时也带来了新的问题：**如果某一次渲染的组件A，B访问了同一个外部资源，而这个资源在A渲染后被更改，然后B再渲染，就导致了视觉上的不一致(visual inconsistent)，不符合直觉。**

### 例子对比
`并发渲染(concurrent rendering)`体现在API上就是通过`startTransition`来触发`setter`函数。

这里通过一个在页面上显示鼠标位置的实验，来更加清晰的展示这种情况。\
当点击`increment`时，手动使页面重新渲染，然后更新页面`x`的值。

```js
import { useState, useCallback, useEffect, useRef, startTransition } from 'react'
import './App.css'

function App() {
  const [count, setCount] = useState(0)
  const increment = useCallback(() => {
    setCount(pre => pre + 1)
  }, [])
  return (
    <div className="App">
      {count} <button onClick={increment}>increment</button>

      {/* 50 * mouseX */}
      {new Array(50).fill(0).map((_, i) => <MouseX key={i} />)}
    </div>
  )
}
// 返回鼠标x的当前位置
function useMouseX() {
  const ref = useRef(0)
  useEffect(() => {
    const onMouseMove = e => {
      ref.current = e.clientX
    }
    document.addEventListener('mousemove', onMouseMove)
    return () => document.removeEventListener('mousemove', onMouseMove)
  }, [])
  return ref.current
}
// 从increment开始，导致的rerender过程中，任何事件都无法被响应，包括用户输入，页面操作
// 所以mousemove产生的事件会被阻塞住，放到队列尾部，当rerender完成时才会被处理
// increment => re-render => mouseX * 50(调用useMouseX函数50次) 
// 但在此过程中，就算鼠标移动了，事件也是被block了，所以ref.current也不会被修改 
function MouseX() {
  const x = useMouseX()
  const now = new Date().getTime()
  while (new Date().getTime() - now < 30) { }
  return (
    <div>
      x: {x}
    </div>
  )
}

export default App

```

![](/images/20220412162824.png)  

可以看道到，就算在渲染期间，鼠标移动 => 触发了`mousemove`事件，但在渲染完成之前都被`阻塞(block)`了，所以不会导致页面的`x`不一致。

### startTransition
但通过`startTransition`将setter函数包裹之后，整个`re-render`都会被拆解成多个`片段`，在执行这些`片段`的中间，可以处理任何事件！所以页面不会再假死，其它事件可以被处理。所以，
> 渲染期间，也可以处理mousemove事件，组件所访问的`x`也可能被修改，导致`visual inconsistent`。

这种现象，在react官方中叫做[tearing](https://github.com/reactwg/react-18/discussions/69)
- [what is tearing?](https://github.com/reactwg/react-18/discussions/69)
```js
function App() {
  const [count, setCount] = useState(0)
  const increment = useCallback(() => {
    // 并发渲染
    startTransition(() => {
        setCount(pre => pre + 1)
    })
  }, [])
  return (
    <div className="App">
      {count} <button onClick={increment}>increment</button>

      {/* 50 * mouseX */}
      {new Array(50).fill(0).map((_, i) => <MouseX key={i} />)}
    </div>
  )
}
```
![](/images/20220412163616.png)

并发渲染后的运行结果显然不符合直觉。
#### hook: useTransition

甚至，在`react18`渲染过程中可以检测到是否渲染完成，未完成时并不会阻塞其它的渲染进程，所以在处理渲染之前，我们可以在页面显示`pending`状态。
网友的例子：
- https://codepen.io/arjunaskykok/pen/xxrvNMN?editors=0110
官方的例子：
- https://17.reactjs.org/docs/concurrent-mode-reference.html#usetransition
### hook: useSyncExternalStore
那么，如何解决这种视觉不一致的问题？所以`react18`也同时提供了一个新的hook：\
`useSyncExternalStore`

> TODO: subscribe, snapshot