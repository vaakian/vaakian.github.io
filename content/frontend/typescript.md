---
title: "Typescript"
date: 2022-01-21T21:15:58+08:00
draft: false
categories: ["TypeScript"]
---

### 动态属性

#### 对象动态属性

即某个对象的属性类型 依赖于 另外一个属性，当Message的type为offer时，payload的属性为`RTCSessionDescriptionInit`，而不是一个联合类型`PeerInfo | RTCSessionDescriptionInit | RTCIceCandidateInit`。
```ts
interface PeerInfo {
  id: string
  nick: string
}
interface PayloadMap {
  join: PeerInfo
  offer: RTCSessionDescriptionInit
  answer: RTCSessionDescriptionInit
  icecandidate: RTCIceCandidateInit
  // leave只有单接受情况，客户端不会主动发送。
  leave: PeerInfo
}

type Message = {
  [K in keyof PayloadMap]: {
    type: K
    nick: string
    receiverId?: string | null
    // playload的类型取决于type的值
    payload: PayloadMap[K]
  }
}[keyof PayloadMap]

```

单拎出来这一段，其实是map出了多个类型，一个key对应一个类型。然后Message通过keyof PayloadMap来获取到一个联合类型。
```ts
const MessageTypeMap = {
  [K in keyof PayloadMap]: {
    type: K
    nick: string
    id: string
    receiverId?: string | null
    // playload的类型取决于type的值
    payload: PayloadMap[K]
  }
}
```


#### 函数动态属性

函数的某个参数类型取决于另外一个参数的类型，与对象不同，函数通过泛型来定义。
```ts
interface Foo {

}
interface Bar {

}
interface eventMap {
  'foo': Foo,
  'bar': Bar
}
type CallbackMap = {
  [K in keyof eventMap]: (event: eventMap[K]) => void
}
/*
CallbackMap等同于
{
    foo: (event: Foo) => void;
    bar: (event: Bar) => void;
}
*/

function add<T extends keyof eventMap>(
  eventName: T,
  cb: CallbackMap[T]
) {
  if(eventName === 'foo') {
    // cb() 接受的第一个参数一定是Foo类型
  } else if(eventName === 'bar') {
    // cb() 接受的第一个参数一定是Bar类型
  }
}

add('foo', (event) => {
  // event现在是Foo类型
})
add('bar', (event) => {
  // event现在是Bar类型
})
```
~~其中T作为泛型，并声明T一定有keyof eventMap当中的属性，当然也可以有其他属性。~~

> **Extends**除了继承，到底能干些什么？如何用？


### 模板字面量类型：Template Literal Types

```javascript
type Animals = 'dog' | 'cat' | 'bird'

type Food = 'meat' | 'vegetable' | 'fruit'

type AnimalsEats = `${Animals} eats ${Food}`
```
结果，类型约束为两个类型的组合。
![](/images/2022-03-11-13-53-57.png)


### 增强原型： Extends prototype
需要通过`interface`来声明新的属性或方法，直接添加会报错。
> interface可以重复声明，会被合并（Merging）。在`TypeScript`中叫做[declaration merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html)。
```typescript
interface Array<T> {
  mapAsync<U>(callbackfn: (value: any, index: number, array: any[]) => Promise<U>, thisArg?: any): Promise<U[]>
}

// 如果不重新声明Array的interface，则会报错，因为Array的interface中没有mapAsync方法。
Array.prototype.mapAsync = function <U>(callbackfn: (value: any, index: number, array: any[]) => Promise<U>, thisArg?: any): Promise<U[]> {
  const promises = this.map(callbackfn.bind(thisArg || this))
  return Promise.all(promises)
}
```


### extends在类型定义中的行为

`T extends U`表示：`T` is assignable to `U`（T可以赋给U），则为真。
当T为联合类型，则会判断每一个的结果，返回联合的结果。

#### 实现Exclude
将T中的U去除掉
```typescript
// T如果是一个union类型，最终结果是遍历的集合
// U extends T 表示 U只能从T里面取
type MyExclude<T, U extends T> = T extends U ? never : T

type result1 = MyExclude<1 | 2 | 3, 2> // 1 | 3
// 相当于以下3条的联合
type result2 = (1 extends 2 ? never : 1)
  | (2 extends 2 ? never : 2) // 这条不满足，取never，然后被删除
  | (3 extends 2 ? never : 3)

```

#### 实现IsIncluded
判断联合类型`T`里面是否有类型`U`，第一个版本如下：
```typescript
type IsIncluded<T, U> = U extends T ? true : false
```
这里有趣的现象发生了，结果可能会出现三种不同的值：

- `true`
- `false`
- `boolean`

```ts
type A = IsIncluded<1 | 2 | 3, 2> // true

// 实际上是 true | true
type B = IsIncluded<1 | 2 | 3, 2 | 3> // true
type C = IsIncluded<1 | 2 | 3, 5> // false

// 实际上是 false | false | false
type D = IsIncluded<1 | 2 | 3, 5 | 6 | 7> // false

// 这条并不会返回false，并不符合逻辑
type E = IsIncluded<1 | 2 | 3, 2 | 4> // boolean!
```

为什么会出现第三种情况`boolean`？\
从`MyExclude`的实现可以看出，实际上`E`的类型等于：
```ts
type Origin = 1 | 2 | 3
type E = (2 extends Origin ? true : false) // true
    | (4 extends Origin ? true : false) // false
```

而`true | false`类型就会被推导为`boolean`类型。

修改方法很简单，只需要判断后面的结果中是否存在`false`，如果满足则最终结果为`false`。
因为`boolean`一定同时包含`true`和`false`，所以也可以直接判断是否为`boolean`。

```ts
// 法1
type IsIncluded<T, U> = false extends (U extends T ? true : false) ? false : true
// 法2
type IsIncluded<T, U> = boolean extends (U extends T ? true : false) ? false : true
// 法3不行，因为：true 也是 assignable to boolean
type IsIncluded<T, U> =(U extends T ? true : false) extends boolean  ? false : true
```


### 手动实现call，并添加类型约束

1. 通过泛型`T`将`context`约束到「原函数」的`this`类型上。
2. 通过泛型`A`将`args`约束到「原函数」的「参数」上。
3. 通过泛型`R`将「函数返回值」约束到「原函数」的「返回类型」上。

```ts
Function.prototype.myCall = function
  <T /* context */,
    A /* args */ extends any[],
    R>
  (this: (this: /*  得到T的类型为this */ T, ...args: A) => R,
    context: T /*  再将context约束到T */,
    ...args: A): R {
  // @ts-ignore
  context[fnKey] = this
  // @ts-ignore
  const result = context[fnKey](...args)
  // @ts-ignore
  delete context[fnKey]
  return result
}
```
如果没有步骤1来约束`context`，那么以下代码将不会报错：


```ts
// 在上面第一个参数中，不取原函数的this类型T
(this: (/* this: T */ ...args: A) => R, ...)

function getName(this: { name: string }, prefix: string) {
    return prefix + this.name
}

// { age: 18 } 正确结果应该是会报错，不存在name: string属性
getName.myCall({ age: 18 }, "name:")
```