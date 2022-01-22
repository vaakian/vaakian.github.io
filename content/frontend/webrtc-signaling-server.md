---
title: "WebRTC: Signaling Server"
date: 2022-01-21T21:20:40+08:00
draft: false
categories: ["WebRTC", "NodeJS"]
---

#### Node.js服务端

服务端所需要做的事情：存储和广播房间信息 & 信令交换中介，包括sdp(SessionDescription)和ice(iceCandidate)。\
接收基本的事件：join, offer, answer, icecandidate, leave

- sdp用于协商会话连接方式之前的必要信息
- ice协商的结果，用于候选的连接方式

~~客户端”仅在“join时带上id和nick~~，然后服务器存储该信息与Client对应。存储方式见：[how to keep track of clients with WebSockets](https://medium.com/@willrigsbee/how-to-keep-track-of-clients-with-websockets-1a018c23bbfc)，或者直接在`server.clients`中挂载信息。\
在offer和answer时，服务器取出对应的id和nick转发出去。

- offer一定是广播，通过`server.clients.forEach`发送
- answer是定向发送，带有receiverId，通过`clientsMap`发送

> client对应表
```js
class ClientsMap {
  constructor(server) {
    this.clients = new Map()
    server.on('connection', (ws, req) => {
      ws.on('message', (message) => {
        const data = JSON.parse(message)
        if (data.type === 'join') {
          this.clients.set(data.id, {
            id: data.id,
            nick: data.nick,
            ws: ws
          })
          // 加入时，先注册好离开事件
          ws.on('close', () => {
            this.clients.delete(data.id)
            this.broadcast(JSON.stringify({
              type: 'leave',
              payload: {
                id: data.id,
                nick: data.nick
              }
            }))
          })
        }

      })

    })
  }

  add(client) {
    this.clients.set(client.id, client)
  }

  remove(client) {
    this.clients.delete(client.id)
  }

  get(id) {
    return this.clients.get(id)
  }

  forEach(callback) {
    this.clients.forEach(callback)
  }
  // 广播offer
  broadcast(senderId, data) {
    const sender = this.get(senderId)
    sender && this.clients.forEach((client, id) => {
      if (id !== senderId) {
        // 向这个client发送
        client.ws.send(data)
      }
    })
  }
  // 定向回复answer
  sendTo(receiverId, data) {
    const receiver = this.clients.get(receiverId)
    if (receiver) {
      receiver.ws.send(data)
    }
  }
}
```

```js
const WebSocket = require('ws')
const wss = new WebSocket.Server({ port: 8080 })
```


> 草稿

file: `signal.js`

```js

const ws = require('ws')

const log = console.log


class SignalEventHandler {
  constructor(signalServer) {
    this.signalServer = signalServer
  }
  join(client, message) {
    // store userInfo
    const userInfo = {
      nick: message.payload.nick,
      id: client.userInfo.id
    }
    client.userInfo = userInfo
    log(`${userInfo.id}  ${userInfo.nick} joined`)
    // or response with room info?
  }
  offer(client, message) {
    this.signalServer.broadcast(client.userInfo.id, {
      ...message,
      // just attach userInfo
      userInfo: client.userInfo,
    })
    log(`${client.userInfo.nick} incoming offer`)
  }
  leave(client) {
    this.signalServer.broadcast(client.userInfo.id, client.userInfo)
  }
  answer(client, message) {
    // data must contain the receiverId
    // client: the offer replier
    this.signalServer.sendTo(message.receiverId, {
      ...message,
      // jus attach userInfo
      userInfo: client.userInfo,
    })
    /*
    {
      type: 'answer',
      data: {sdp, type},
      receiverId: 'xxx',
      userInfo: {id, nick},
    }
    */
    log(`${client.userInfo.nick} incoming answer(reply offer)`)
  }
  candidate(client, message) {
    this.signalServer.broadcast(client.userInfo.id, {
      ...message,
      // just attach userInfo
      userInfo: client.userInfo,
    })
    log(`${client.userInfo.nick} incoming candidate`)
  }
  // this is bind to ws
  handle(client, message) {
    try {
      message = JSON.parse(message)
      // {type, receiverId, payload}
      this[message.type](client, message)
    } catch (err) {
      log(err)
    }
  }
}

class SignalServer {
  constructor() {
    this.server = new ws.WebSocketServer(...arguments)
    this.server.on('connection', (ws, req) => {
      ws.userInfo = { id: req.headers['sec-websocket-key'] }
      // log('client connected', req.headers['sec-websocket-key'])
      const signalEventHandler = new SignalEventHandler(this)
      ws.on('message', function (message) {
        signalEventHandler.handle(ws, message)
      })
      // leave event on connection
      ws.on('close', function () {
        signalEventHandler.leave(ws)
      })
    })
  }
  on() {
    this.server.on(...arguments)
  }

  broadcast(senderId, data) {
    this.server.clients.forEach((client) => {
      if (senderId !== client.userInfo.id) {
        client.send(JSON.stringify({
          ...data,
          senderId: senderId
        }))
      }
    })
  }
  sendTo(receiverId, data) {
    this.server.clients.forEach(client => {
      if (client.userInfo.id === receiverId) {
        client.send(JSON.stringify(data))
      }
    })
  }
}
// const srv = SignalServer({ port: 666 })

module.exports = {
   serve(options) {
    return new SignalServer(options)
  }
}
```

file: `index.js`


```js

const SignalServer = require("./signal")

const srv = SignalServer.serve({ port: 666 })
```

