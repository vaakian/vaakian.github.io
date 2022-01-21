---
title: "Typescript"
date: 2022-01-21T21:15:58+08:00
draft: flase
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
  leave: PeerInfo
}

type Message = {
  [K in keyof PayloadMap]: {
    type: K
    nick: string
    id: string
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