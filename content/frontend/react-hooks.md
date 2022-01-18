---
title: "React Hooks"
date: 2022-01-18T14:47:17+08:00
draft: false
categories: ["React", "Frontend", "JavaScript"]
---

[TOC]

## useMemo

千万不要把useMemo和React.memo混为一谈，两者都是根据依赖变化来缓存数据的。`useMemo(fn, deps)`参照第二个参数列表的变化返回新的数据，否则返回缓存的数据。而`React.memo`只会在组件的props变化时重新渲染该组件。

### 使用场景
- 有非常耗时的计算，比如以下例子中的`expensiveComputation`，每一次组件渲染，都会调用该函数，并且重新计算。但其实a和b并没有变化时，计算的值都是一样的，所以需要使用到useMemo将结果缓存下来，而无需重新计算。
```js
function App() {
    const [a, setA] = useState(1)
    const [b, setB] = useState(2)
    // const value = expensiveComputation(a, b)
    const value = useMemo(() => expensiveComputation(a, b), [a, b])
    return <div>result: {value}</div>

}
```
- 任何**hook**的依赖为对象时，可能会遇到「字面上」相同，但不是同一个对象，所以会导致依赖判断错误。因为**hook**的依赖数组对比是浅层的对比，也就是 `===` 对比。因为每次渲染时，组件内的所有字面变量都会被重新创建，所以遇到对象应该要格外注意。
> 对象判定相等
```js
const obj = {count: 1}
console.log(obj === obj) // true

// 引用不一样，所以为false
console.log({count: 1} === {count: 1}) // false
```


以下例子，effect中的依赖是对象，虽然a, b没有发生变化，obj的字面上看起来也没用变化。但是在每次渲染时，都会创建一个新的对象，该对象引用不一样，所以每次都会执行effect。
> hook依赖例子
```js
function App() {
    const [a, setA] = useState(1)
    const [b, setB] = useState(2)
    const obj = {a, b}
    useEffect(() => {
        console.log('obj changed')
    // 当obj发生变化时，执行effect
    }, [obj])
    })
    return <div>App</div>
}
```
用useMemo解决该问题
```js
function App() {
    const [a, setA] = useState(1)
    const [b, setB] = useState(2)
    // 当a,b发生变化时，创建新的obj
    const obj = useMemo(() => {
        return {a, b}
    }, [a, b])

    useEffect(() => {
        console.log('obj changed')
    // 当obj发生变化时，执行effect
    }, [obj])
    })
    return <div>App</div>
}
```


## useCallback

useCallback也是解决类似useMemo的引用问题，但不同的是它缓存函数。useCallback的功能也可以通过useMemo来实现。\
当子组件根据props的变化，调用父组件的方法或者进行任何变化判定时，父组件的方法可能会被重复调用，而useCallback可以缓存调用过的方法，从而避免重复调用。
```js
// 当props发生变化才重新渲染，否则复用最近的一次渲染结果。
const Child = React.memo(props => {
    const { onButtonClick } = props
    return <button onClick={onButtonClick}>click</button>
})

function App() {
    const handleClick = () => {
        console.log('Clicked!')
    }
    return (
      <div>
        <Child onButtonClick={handleClick} />
      </div>
    )
}
```
以上的handleClick看似正常，虽然被React.memo包裹，但Child仍然会被重新渲染。因为每次渲染handleClick都是一个新创建的函数，Child的`props`会被判定发生变化。所以使用useCallback可以解决这类问题。
```js
function App() {
    const handleClick = useCallback(() => {
        console.log('Clicked!')
    }, [])
    return (
      <div>
        <Child onButtonClick={handleClick} />
      </div>
    )
}
```
useCallback的第二个参数为依赖，在该回调依赖的变量发生变化时再重新创建函数。由此可见，useCallback和useMemo的功能极为相似。可以说：
```js
useCallback(fn, deps)
// 相当于
useMemo(() => fn, deps)
```

## useContext


## useReducer

