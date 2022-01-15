---
title: "论文草稿：WebRTC--RTCPeerConncetion"
date: 2022-01-15T13:29:47+08:00
draft: false
categories: ["论文"]
---

## what is WebRTC

WebRTC(Web Real-Time Communication)是一个最先由Google发布的点对点（Peer-to-Peer）通信协议，目前已经被各大浏览器所内置，早期不同浏览器的API有细微差别，需要开发者手动做兼容。现在可以使用MDN所推荐的[WebRTC adapter.js](https://github.com/webrtc/adapter/)来解决兼容问题。\
其次它不仅仅适用于浏览器，也可以在任何其它不同平台的应用程序中使用，只要符合WebRTC的规范即可。


## 信道建立流程

两个实体（Peer）要建立连接或者说信道之前，需要先进行一项最最重要的工作，即信令交换。它将两个实体的信息（包含IP地址、端口等等各种用于建立连接的必要信息）进行互相交换，双方拿到信息后建立独立的信道。此后，两个实体就可以通过信道进行通信了，无需再通过服务器进行数据传输。

## 两种通信方式的优缺点

点对点通信方式不同于传统的中心化通信方式，需要一台或者多台服务器进行数据转发或者存储，这样会导致服务器的带宽和计算压力增大，而当服务器的负载增加时，会进而导致整个系统的稳定性下降，一旦中心服务器宕机，所有用户的通信将全部中断。而如果采用点对点通信方式，省去了服务器中转存储环节，降低了各个实体之间依赖性。用户之间可以直接通过网络进行通信，某个实体中断仅仅影响它自身，这样既可以节省服务器的资源，又提高了系统的稳定性。\
不过，点对点通信也有自身的缺点，由于视频会议属于多对多的连接，每个实体要和其它所有实体都建立连接，随着系统中实体的增加，单个实体的压力也会随之增加。而采用的中心服务器转发存储方式就不会出现这种问题， 每个实体只需要和中心服务器建立连接，中心服务器负责存储数据和转发数据到其它实体。\
所以，在实体较少、传输数据量大、网络延迟要求更高的场景下，点对点通信方式比较适合，反之，中心服务器转发存储方式比较适合。

## RTCPeerConnection

RTCPeerConnection负责很多事情，包括：

- 信令处理（singal processing）
- 编解码（Codec handling）
- 点对点连接（P2P connection）
- 安全（Security）
- 流量控制（bandwidth  management）
- ……

RTC创建连接过程主要是通过一个信令服务器进行SDP信息交换，这个过程可以通过WebSocket进行，也可以通过HTTP进行，当然WebSocket最佳。
```js
const pc = new RTCPeerConnection(null)

function sendOffer(desc) {
    // 发送offer到服务器上
    ws.send({type: 'offer', sdp: desc})
    .then(gotAnswer)
}
// 当成功创建offer时调用
pc.createOffer(gotOffer)
function gotOffer(desc) {
    pc.setLocalDescription(desc)
    // 发送到服务器
    sendOffer(desc)
}
// 当服务器返回answer时调用
function gotAnswer(desc) {
    // 当设置远程描述（SDP）之后，WebRTC就会开始创建连接
    pc.setRemoteDescription(desc)
}

// 错误处理：断开连接，webSocket连接优先判定
function handleError(err) {
    console.log(err)
    ws.close()
}
```
首先Peer进入页面，会创建一个RTCPeerConnection，随后发送一个offer给服务器，服务器将该offer广播至其它Peer，其它Peer再回复一个answer给该Peer。\
也就是说，`仅当收到后Answer，才会开始创建连接。`

模拟流程：
- A进入房间，创建offer，发送到服务器，服务器广播给其它用户。
- 若房间为空，则不会有下一步操作。
- 若房间有人，遍历发送offer给每个人，然后每个人会回复一个answer给服务器。
- 服务器将answer返回给A，A开始创建连接。

由于每一个peer会有一个id，所以服务器可以通过该id来确定哪个peer发送的offer，然后定向返回answer。


![](/images/webrtc1.png)
