---
title: "React之：自定义hooks: useCamera"
date: 2021-12-14T15:55:03+08:00
draft: false
categories: ["React", "Frontend"]
---
尝试用浏览器API封装一个读取摄像头视频流的useCamera自定义hooks，一步一步优化，总结一下得到目前为止的最佳实践。
首先，摄像头读取API需要传入最基本的参数[constraints](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia#parameters)，通过promise方式得到stream后会展示到video标签上，那么`useCamera`应该接受一个能够读取到video标签的参数，那么首选ref，得到如下第一版代码：

```jsx
function useCamera(constraints, videoRef) {
  const storedStream = useRef(null)
  // 当stream改变时，创建新的stop函数
  const stop = useCallback(() => {
    storedStream.current.getTracks().forEach(track => track.stop())
  }, [storedStream.current])

  // 当constraints/videoRef改变时，创建新的start函数
  const start = useCallback(() => {
    navigator.mediaDevices.getUserMedia(constraints)
      .then(stream => {
        console.log('setting stream')
        videoRef.current.srcObject = stream
        storedStreamRef.current = stream
      })
      .catch(err => {
        console.error(err)
      })
  }, [constraints, videoRef])
 // constraints改变时，stop上一次的媒体流并重新请求
  useEffect(() => {
    start()
    return stop
  }, [constraints])
  return [start, stop]
}
```

然后在组件中调用创建的hooks，功能上运行正常。
```jsx
function App() {
    const videoRef = useRef(null)
    const [start, stop] = useCamera({ video: true, audio: false }, videoRef)
    return (
    <div className="App">
      <video ref={videoRef} autoPlay></video>
      <button onClick={stop}>stop</button>
      <button onClick={start}>start</button>
    </div>
  )
}
```

### 依赖变化：对象深比较
但发现当页面`re-render`的时候，`video`视频标签会出现闪屏，明显是`video`标签的视频流被重新设置了。但`useEffect`的`constraints`是一个字面量，怎么会导致依赖变化然后执行*side effect*呢？
后来经过仔细调试发现是因为字面量每次都会重新创建，虽然内容一样，但实际上并不是同一个对象，而useEffect是浅比较，也就是“===”号比较，所以导致判定依赖发生变化。解决办法有很多种：

- 将constraints对象写在App组件之外，则不会重新创建：**不够内聚**
- 通过JSON.stringify转变为字符串来进行比较：**土办法，性能差**
- 通过useState来存储constraints，只要没有调用set方法，那么useState将返回原对象：**代码多了，比较麻烦**
- 通过useMemo来存储constraints，依赖留空数组：**同上**
- 自定义一个`useDeepCompareEffect` hooks：**最佳实践，这样就可以写字面量了**




```js
function useDeepCompareEffect(callback, dependencies) {}

```


### 优化hooks设计逻辑
useCamera需要接受一个ref来访问video标签，这其实增加了的耦合性，更好的方式是将媒体流通过hooks返回，交给开发者自由使用。其二，访问摄像头需要用户同意，那么也应该返回摄像头访问权限状态：请求中、同意、拒绝、已停止和媒体设备情况（摄像头、麦克风）。

那么代码如下

```jsx
function App() {
  const videoRef = useRef(null)
  const [stream, status, start, stop] = useCamera({ video: true, audio: false })

  useEffect(() => {
    if (status === 'success') {
      videoRef.current.srcObject = stream
    }
    return () => {
      videoRef.current.srcObject = null
    }
  }, [status])
  return (
    <div className="App">
      <div>status: {status}</div>
      <video ref={videoRef} autoPlay></video>
      <button onClick={stop}>stop</button>
      <button onClick={start}>start</button>
    </div>
  )
}
```

hooks实现，那些变量需要用到useState？哪些方法需要用到useCallback？
显然，选择status最佳，当status变化时，页面需要作出不同的响应。

> 如果不用useState来存储status，那么当status变化时，不会触发rerender，进而useEffect不会比较status并作出响应。
```js

function useCamera(constraints) {
  const storedStream = useRef(null)
  const [status, setStatus] = useState('pending')
  // 当stream改变时，创建新的stop函数
  const stop = useCallback(() => {
    storedStream.current.getTracks().forEach(track => track.stop())
    setStatus('stoped')
  }, [storedStream.current])

  // 当constraints/videoRef改变时，创建新的start函数
  const start = useCallback(() => {
    setStatus('pending')
    navigator.mediaDevices.getUserMedia(constraints)
      .then(stream => {
        console.log('setting stream')
        storedStream.current = stream
        setStatus('success')
      })
      .catch(err => {
        setStatus('error')
        console.error(err)
      })
  }, [constraints])

  useDeepCompareEffect(() => {
    start()
    return stop
  }, [constraints])
  return [storedStream.current, status, start, stop]
}
```


### 使用hooks时记住一句话

> 先导致rerender，才会有新的compare，从而出现一系列effect。