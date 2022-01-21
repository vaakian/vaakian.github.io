---
title: "WebRTC: Multi-RTCPeerCoonection"
date: 2022-01-21T16:24:57+08:00
draft: false
categories: ["frontend", "WebRTC"]
---

## 一）抛出问题

一个RTCPeerConnection只能对应另外一个RTCPeerCoonection，如果想要实现多人会议。那么需要统一管理多个RTCPeerConnection，任何本地数据都要广播到多个RTCPeerConnection中去。

## 二）职责分割

在通过纯原生的RTCPeerConnection实现双人一对一视频会议时，发现代码分隔时已经力不从心了。如何实现多人更需要一些明确的职责分割。

- 管理多个PeerConnection
- 将本地数据广播到多个PeerConnection
- ***如何将PeerConnection与WebSocket中的Client绑定（如何识别身份）**
- Track概念：管理多个Track到一个Stream中
- 本地Stream推到远程

### 2.1 如何识别身份

> 如何在WebSocket和PeerConnection建立连接过程中去确定身份，如何确定两者相关联？ 

首先要理清楚，建立连接的整个流程。然后找出应该在哪个阶段去进行身份确认。

1. 加入房间，发送JoinRoom消息，带上自己的nick和id（唯一标识），服务器存储信息对应，回复房间Clients列表。
2. 如果房间有人，发送Offer消息（带nick和id），服务器广播之后，其他人回复Answer消息（带个人信息和将要发往的id）
3. 前端接收到Answer，将对应的PeerConnection与该id和nick对应。此后的连接就容易对应了。
![WebRTC Signaling](/images/webrtc-signaling.png)
```ts
peerConnection.createOffer()
// 1. 创建offer
  .then(offer => {
    return peerConnection.setLocalDescription(offer);
  })
  .then(() => {
// 2. 发送offer（广播）
    return ws.send(JSON.stringify({
      type: 'offer',
      nick: nick,
      id: id,
      sdp: peerConnection.localDescription
    }));
  })
  .then(() => {
    return new Promise((resolve, reject) => {
// 3. 接收answer
      ws.onmessage = (event) => {
        const data = JSON.parse(event.data);
        if (data.type === 'answer') {
          resolve(data);
        }
      };
    });
  })
  .then(data => {
// 4. 处理answer
    return peerConnection.setRemoteDescription(data.sdp);
  })
  .then(() => {
    console.log('连接建立完成');
  })
  .catch(error => {
    console.error(error);
  });

```

#### 定义Peer结构

一个远程id对应一个本地RTCPeerConnection，多个Peer就存入一个Peers数组。
```ts
interface Peer {
    id: string;
    nick: string;
    // LocalPeer与该Peer连接的RTCPeerConnection
    peerConnection: RTCPeerConnection;
    // track有两种，屏幕共享和用户音视频（远程向本地共享的）
    media: {
        display: MediaStreamTrack[];
        user: MediaStreamTrack[];
    }
    // dataChannel用于发送文本或其它二进制数据（文件）
    dataChannel: RTCDataChannel | null;
}


// 一个LocalPeer对应对个Peers
interface LocalPeer {
    id: string;
    nick: string;
    Peers: Peer[];
}

```
### 2.2 建立流程 & 事件处理

> TODO: 整个WebSocket+RTCPeerConnection的连接过程，都先不要用hooks来写，太麻烦。后面暴露出几个重要的方法，然后通过简单的hooks二次封装简化编码流程。

#### 1> 定义事件处理器（eventHandlers）

先定义数据交换模型
```ts
interface PeerInfo {
    id: string;
    nick: string;
}
interface PayloadMap {
  join: PeerInfo;
  offer: RTCSessionDescriptionInit;
  answer: RTCSessionDescriptionInit;
  icecandidate: RTCIceCandidateInit;
  leave: PeerInfo;
}
// 客户端发送，服务端接受的数据格式
type Message = {
    [k in keyof PayloadMap]: {
        type: k;
        nick: string;
        id: string;
        receiverId?: string | null;
        // playload的类型取决于type的值
        payload: PayloadMap[k];
    }
}[keyof PayloadMap]

```

##### 2> 处理offer

```js
function gotOffer(offer: RTCSessionDescriptionInit) {
    // 本地成功创建offer
}

function onOffer(offer: RTCSessionDescriptionInit) {
    // 接收到远程发来的offer
}
```
##### 3> 处理answer


##### 4> 处理icecandidate



### 2.2 什么是track

track英文`/træk/`译为`轨道`，在WebRTC中，Track同样是一个轨道，可以是音频轨道，也可以是视频轨道。早期的WebRTC版本PeerConnection通过AddStream()方法添加Stream，而不是track。后来的版本，PeerConnection通过addTrack()方法来传递音视频轨道，这样做的好处就是，接收者可以通过track.kind来判断是音频轨道还是视频轨道，然后选择是否接收该track。接收者可以单独接收音频轨道和单独接收视频轨道。而AddStream()的方式不能够选择性的接收。
```js
const remoteStream = new MediaStream()
pc.addEventListener('track', (event) => {
  // 选择性接受
    if (event.track.kind === 'video') {
        // 接收视频
        remoteStream.addTrack(event.track)
    } else if (event.track.kind === 'audio') {
        // 接收音频
        // remoteStream.addTrack(event.track)
    }  
})
```