---
title: "Publish Ts Package"
date: 2022-01-25T17:37:56+08:00
draft: true
categories: ["npm"]
---

> 草稿

由publish最近写的`XPeer`包引发的一系列新问题，记录学习所得。

`XPeer`是用TypeScript来写的，可以正常publish到npm上，但只能被ts项目所引用，而且需要反复编译，会浪费不少资源。\
[The 30-second guide to publishing a TypeScript package to NPM](https://cameronnokes.com/blog/the-30-second-guide-to-publishing-a-typescript-package-to-npm/)

## 预编译

### tsc project
仅仅是将ts编译出原生js，前端项目能够直接import，不管是js还是ts项目。但不能通过浏览器`<script>`标签来引入。

### browserify
browserify能够对js进行bundle操作，让浏览器能够直接引入。至于什么是bundle？
`tinify`是browserify的压缩package，可以让bundle瘦身。
`browserify-global-shim`让某个包可以在windiw对象上全局访问。

## 配置

## publish流程
npm init
配置package.json
"main": "lib/index.js",即import的默认入口
"types": "lib/index.d.ts", 待考察
"exclude"
"include"
"declaration" 如何管理d.ts文件和types属性，让引包用户能够正常拿到type？

npm login

npm publish

## 使用

> npm i -s xpeer

```js
import { XPeer } from 'xpeer'
```

## 总结