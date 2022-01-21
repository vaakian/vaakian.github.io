---
title: "Typescript"
date: 2022-01-21T21:15:58+08:00
draft: flase
categories: ["TypeScript"]
---

### 动态属性
即某个属性类型 依赖于 另外一个属性，当Message的type为offer时，payload的属性为`RTCSessionDescriptionInit`，而不是一个联合类型`PeerInfo | RTCSessionDescriptionInit | RTCIceCandidateInit`。
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