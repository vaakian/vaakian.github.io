---
title: 'write a simple Vite plugin'
date: 2022-05-10T14:50:32+08:00
draft: false
categories: ['Vite']
---

## prerequisites

在 `webpack` 中，分为 plugin 和 loader，其中 loader 的职责被划分为单一的`翻译`转换。
而在 `vite` 中，去除了 loader 的概念，而是将代码转换/编译的概念统一放到了 plugin 中去。

## plugin

[vite: using plugin](https://vitejs.dev/guide/using-plugins.html)

### 目的

理解最基本的 plugin 概念，并为 vite 编写一个最简单的 plugin。

### 任务：创建一个 plugin 将以下文件进行编译，并实现`import`

文件：hello.wj

```js
{
    "name": "John",
    "age": 12
}
----
<div>{ name }</div>
<div style="color: green; font-size: 24px;">{ age }</div>
```

编译为

```html
<div>John</div>
<div style="color: green; font-size: 24px;">12</div>
```

在 js 文件中引入

```js
import content from './hello.wj';
console.log(content);
/*
<div>John</div>
<div style="color: green; font-size: 24px;">12</div>
*/
```

### 实现

在 vite 中，plugin 接受一个对象，包含很多属性，这里用到`name`和`transform`，transform 是一个函数，接受 code（文件内容）和 id（文件绝对路径）。如果 transform 返回值不为空，则这时候将返回的值作为编译后的代码，否则将返回原始的 code。

与 webpack 一样，plugin 在数组中的位置也很重要，因为下一个`transform`接收到的是上一个`plugin`处理后的结果。

```js
import { defineConfig } from 'vite';
export default defineConfig({
  plugins: {
    name: 'WJPlugin',
    transform(code, id) {
      if (!id.endsWith('.wj')) return;
      // 实现：转换功能
      return wjCompiler(code);
    },
  },
});
```

### 实现 wjCompiler

将数据和模板通过"----"进行分割，然后将模板和数据合并，最后返回结果。但注意返回的结果要通过 export default `string`导出。\
**因为通过 import 方式导入的任何内容，一定要是 javascript 代码。**，比如通过 vite 导入 css 文件，最终会转换成代码功能：创建一个 style 标签，并将 css 内容放入。

```ts
export function wjCompiler(code: string) {
  const [data, template] = code.split('----');
  const serializedData = JSON.parse(data);
  const rawHtml = template
    .trim()
    .replace(/\{\s+(.*)\s+\}/g, (_, property) => serializedData[property]);
  // 生成可执行的js代码
  return `export default \`${rawHtml}\``;
}
```

### 编写简单的单元测试

![unit test](/images/20220510151254.png)

### 简单的插入到页面中

代码：通过 import 直接得到了编译后的结果，然后插入到页面中。
![web page](/images/20220510151755.png)
页面：

![](/images/20220510151908.png)

## 动态 plugin

现在的插件，一般都会封装成一个函数去动态创建 plugin 对象，而不是直接提供一个静态 plugin。这样做的好处是可以通过函数参数去对 plugin 进行配置。

> wjPlugin.ts

```js
import { PluginOption } from 'vite';

export function WJPlugin(config: { enabled: boolean }): PluginOption {
  return {
    name: 'WJPlugin',
    transform(code, id) {
      if (!id.endsWith('.wj')) return;
      // 读取配置
      if (!config.enabled) return;
      return wjCompiler(code);
    },
  };
}
}
```

那么在使用的时候，就可以插入配置。

```ts
import { defineConfig } from 'vitest';
import { WJPlugin } from './plugin';
export default defineConfig({
  plugins: [
    WJPlugin({
      enabled: false,
      //   enabled: true,
    }),
  ],
});
```

## TODO

Virtual Modules 是一个非常有用的`technique`，可以引入一个虚拟模块，然后通过 plugin 的`resloveId`和`load`方法插入动态 js 代码。

> Virtual modules are a useful scheme that allows you to pass build time information to the source files using normal ESM import syntax.

- [Virtual Modules](https://vitejs.dev/guide/api-plugin.html#importing-a-virtual-file)

### usage

```ts
import something from 'virtual:my-module';
console.log(something); // works!
```
