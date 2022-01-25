---
title: "如何在WebRTC中确定Track的类型（摄像头/屏幕）"
date: 2022-01-25T15:37:52+08:00
draft: true
---
本地向远程端推多个track时（摄像头，屏幕），如何去区分该track的类型。
```js

```

修改SDP，添加track类型。在接收端，trackEvent拿到remoteDescription，然后在sdp中匹配类型。

```js
event.target.remoteDescription.sdp.toString()
```

