---
title: "关于 tree-shaking"
date: 2022-09-15T21:15:58+08:00
draft: false
categories: ["Frontend"]
---


最近对Tree-shaking做了不少的了解和研究，这里做一些个人总结，方便以后遗忘时进行复习。

## 什么是Tree-shaking



### 术语解释
Tree-shaking是一种用于消除JavaScript上下文中的未引用代码(dead-code)的优化技术。它是一种基于ES6模块的静态代码分析技术，可以在编译时就进行代码的消除，从而减少代码体积。

### 通俗解释
从英文字面意思“摇树”上来看，就是把一棵树上的不需要的叶子摇下来，只保留需要的叶子。在编程语言中，Tree-shaking就是指在打包的时候，把不需要的代码摇下来，只保留需要的代码。

## Tree-shaking的原理

早期大部分node包都是commonJS格式，也就是通过require来引入。它有一个致命的缺陷：因为它的动态性，无法在编译时就确定依赖关系，所以很难找到不用的代码进行tree-shaking。

到了ESModule时代，Tree-shaking变得容易起来。\
tree-shaking概念最早是由Rollup.js提出的，它就是利用ESM的静态优势，通过静态分析找出模块之间的依赖关系然后把没有用到的代码去掉。

## 那么早期没有ESM，tree-shaking技术的时候，怎么做到按需打包？

在Tree-shaking没有兴起的时候，前端库在打包的时候可以不进行bundle，每个组件都保留成单独的文件，引入的时候也需要按照文件目录引入。也就是通过“文件隔离”来达到静态性。这种方式的缺点是：引入的路径会非常长，增加了学习成本，不够优雅。

```js
import { Button } from 'antd/lib/button'
import { Table } from 'antd/lib/table'
```

所以后来前端库在按需引入时会同时提供一个插件，它会帮你自动匹配文件目录，然后引入。显而易见这种方式的缺点是：需要在项目中配置额外的插件。
```js
import { Button } from 'antd'
// 通过插件自动转换成👇
var Button = require('antd/lib/button')
```

> 不管配置插件有多简单，只要多一个步骤，总会产生新的问题，给开发者增加学习成本。

可以打开 https://github.com/umijs/babel-plugin-import 看看，相信你也无法在10分钟内看懂它的用法并成功跑起来，这就是多余的学习成本。

## 在ESM时代，也同时进入了tree-shaking的时代

好在有ESM的天然静态优势加持，我们现在可以既bundle，又保留tree-shaking的能力。这一切对于使用库的开发者来说，都是自动实现的，没有额外的学习成本。

[Rollup官网示例：tree-shaking](https://rollupjs.org/repl/?version=2.79.1&shareable=JTdCJTIybW9kdWxlcyUyMiUzQSU1QiU3QiUyMm5hbWUlMjIlM0ElMjJtYWluLmpzJTIyJTJDJTIyY29kZSUyMiUzQSUyMiUyRiolMjBUUkVFLVNIQUtJTkclMjAqJTJGJTVDbmltcG9ydCUyMCU3QiUyMGN1YmUlMjAlN0QlMjBmcm9tJTIwJy4lMkZtYXRocy5qcyclM0IlNUNuJTVDbmNvbnNvbGUubG9nKCUyMGN1YmUoJTIwNSUyMCklMjApJTNCJTIwJTJGJTJGJTIwMTI1JTIyJTJDJTIyaXNFbnRyeSUyMiUzQXRydWUlN0QlMkMlN0IlMjJuYW1lJTIyJTNBJTIybWF0aHMuanMlMjIlMkMlMjJjb2RlJTIyJTNBJTIyJTJGJTJGJTIwbWF0aHMuanMlNUNuJTVDbiUyRiUyRiUyMFRoaXMlMjBmdW5jdGlvbiUyMGlzbid0JTIwdXNlZCUyMGFueXdoZXJlJTJDJTIwc28lNUNuJTJGJTJGJTIwUm9sbHVwJTIwZXhjbHVkZXMlMjBpdCUyMGZyb20lMjB0aGUlMjBidW5kbGUuLi4lNUNuZXhwb3J0JTIwZnVuY3Rpb24lMjBzcXVhcmUlMjAoJTIweCUyMCklMjAlN0IlNUNuJTVDdHJldHVybiUyMHglMjAqJTIweCUzQiU1Q24lN0QlNUNuJTVDbiUyRiUyRiUyMFRoaXMlMjBmdW5jdGlvbiUyMGdldHMlMjBpbmNsdWRlZCU1Q25leHBvcnQlMjBmdW5jdGlvbiUyMGN1YmUlMjAoJTIweCUyMCklMjAlN0IlNUNuJTVDdCUyRiUyRiUyMHJld3JpdGUlMjB0aGlzJTIwYXMlMjAlNjBzcXVhcmUoJTIweCUyMCklMjAqJTIweCU2MCU1Q24lNUN0JTJGJTJGJTIwYW5kJTIwc2VlJTIwd2hhdCUyMGhhcHBlbnMhJTVDbiU1Q3RyZXR1cm4lMjB4JTIwKiUyMHglMjAqJTIweCUzQiU1Q24lN0QlMjIlMkMlMjJpc0VudHJ5JTIyJTNBZmFsc2UlN0QlNUQlMkMlMjJvcHRpb25zJTIyJTNBJTdCJTIyZm9ybWF0JTIyJTNBJTIyZXMlMjIlMkMlMjJuYW1lJTIyJTNBJTIybXlCdW5kbGUlMjIlMkMlMjJhbWQlMjIlM0ElN0IlMjJpZCUyMiUzQSUyMiUyMiU3RCUyQyUyMmdsb2JhbHMlMjIlM0ElN0IlN0QlN0QlMkMlMjJleGFtcGxlJTIyJTNBJTIyMDIlMjIlN0Q=)

## ⚠️注意事项
tree-shaking的粒度，以被export的资源为单位。\
在开发你的库时，你需要注意以下几点：

### 1. 注意函数的副作用

```js
// utils.js

export const zero = 0

export const getValue = (a) => {
  return a.value
}

// 这里有副作用
export const five = getValue({value: 5})
```

就算没有引入five，但是getValue函数会被执行，这就是副作用。为了保证shake后的结果/行为一致，就必须要保留这段代码函数。

为什么？因为这个函数可能会有副作用，比如说修改全局变量，或者是修改DOM，或者是发送网络请求等等。然后其它地方的代码可能会依赖这些被修改的数据。

```js
// main.js
import { zero } from './utils'

console.log(zero)
```

所以最后的产物输出为：

```js
const zero = 0

const getValue = (a) => {
  return a.value
}

getValue({value: 5})

console.log(zero)
```
![](/images/20220924181410.png)  

要避免这种情况，就需要告诉bundler，这个函数没有副作用。目前有两种方式实现：

#### 1.1 pacakge.json中的 `sideEffects`属性



```json
// pacakge.json
{
  "name": "myPkg",
  "sideEffects": false
}
```

https://webpack.js.org/guides/tree-shaking/#mark-the-file-as-side-effect-free

这种方式最早由webpack提出，但是后来被rollup也支持了。这种方式的优点是简单，缺点是粒度太粗，只能针对整个包进行设置。如果你的包中有一些函数是有副作用的，而另一些函数是没有副作用的，那就要用到第二种方法。

#### 1.2 魔法注释（magic comment）

通过 `/*#__PURE__*/` 注释告诉bundler，这个函数没有副作用。


```js
// utils.js

export const zero = 0
export const getValue = (a) => {
  return a.value
}

export const five = /* #__PURE__ */ getValue({value: 5})
```

![](/images/20220924182201.png)  

### 2. 不要用「对象」或者「Class」等等任何动态变量来封装模块

将所有你认为需要按需的资源，都通过export暴露出来。一定不要用对象或者Class等等任何动态变量来封装模块。因为前面说过，tree-shaking的粒度是以被export的资源为单位。


[查看rollup在线示例](https://rollupjs.org/repl/?version=2.79.1&shareable=JTdCJTIybW9kdWxlcyUyMiUzQSU1QiU3QiUyMm5hbWUlMjIlM0ElMjJtYWluLmpzJTIyJTJDJTIyY29kZSUyMiUzQSUyMmltcG9ydCUyMHRvb2xzJTIwJTIwZnJvbSUyMCcuJTJGdXRpbHMnJTVDbiU1Q25jb25zb2xlLmxvZyh0b29scy5mb28pJTIyJTJDJTIyaXNFbnRyeSUyMiUzQXRydWUlN0QlMkMlN0IlMjJuYW1lJTIyJTNBJTIydXRpbHMuanMlMjIlMkMlMjJjb2RlJTIyJTNBJTIyZXhwb3J0JTIwZGVmYXVsdCUyMGNsYXNzJTIwdG9vbHMlMjAlN0IlNUNuJTIwJTIwc3RhdGljJTIwZm9vKCklMjAlN0IlNUNuJTIwJTIwJTIwJTIwY29uc29sZS5sb2coJ2ZvbycpJTVDbiUyMCUyMCU3RCU1Q24lMjAlMjBzdGF0aWMlMjBiYXIoKSUyMCU3QiU1Q24lMjAlMjAlMjAlMjBjb25zb2xlLmxvZygnYmFyJyklNUNuJTIwJTIwJTdEJTVDbiU3RCUyMiUyQyUyMmlzRW50cnklMjIlM0FmYWxzZSU3RCUyQyU3QiUyMm5hbWUlMjIlM0ElMjJtb2R1bGVfMS5qcyUyMiUyQyUyMmNvZGUlMjIlM0ElMjIlMjIlN0QlNUQlMkMlMjJvcHRpb25zJTIyJTNBJTdCJTIyZm9ybWF0JTIyJTNBJTIyZXMlMjIlMkMlMjJuYW1lJTIyJTNBJTIybXlCdW5kbGUlMjIlMkMlMjJhbWQlMjIlM0ElN0IlMjJpZCUyMiUzQSUyMiUyMiU3RCUyQyUyMmdsb2JhbHMlMjIlM0ElN0IlN0QlN0QlMkMlMjJleGFtcGxlJTIyJTNBbnVsbCU3RA==)

![](/images/20220924191529.png)  
改造后，未用到的`bar`函数被成功“shake”掉了。
![](/images/20220924191449.png)  

### 3. 如果产物是单个bundle，那么注意不要封装子模块，否则会失去tree-shaking的能力。

因为在同一个文件下，ESM并没有提供静态的导出子模块的能力。也就是并没有提供大概以下的语法：

```js
const foo = 'foo'
const bar = 'bar'

const a = 1
const b = 2

export { foo, bar } as subModule1
export { a, b } as subModule2
```
所以打包工具（rollup/webpack）在bundle的时候，会将子模块封装到一个对象中，然后再导出。这样整个子模块就变成了一个被exprot的变量，它的shake粒度就是整个子模块，而不是子模块中的某个函数。那么开发者在使用你这个模块的时候，自然就没有了按需引入的能力。

![](/images/20220924184018.png)
如图所示，我们将output产物作为一个包引入，最后得到如下结果：\
我们只用到了`sub1.foo`函数，而sub1被整个打包进output产物了。
![](/images/20220924184304.png)  
### 4. 代码被 `bundle` / `打包` 的时机

在当今的前端工程化中，我们的代码最终会被各种loader、plugin、打包工具等等处理，将一类代码编译转换成另外一类代码，比如ts => js，esm => cjs，或者是通过做babel兼容。

![](/images/20220924185531.png)

如上图的编译结果，虽然我们通过export静态导出了一个class，但是在最终的代码中，我们发现它被打包成了一个对象，然后再通过`module.exports`导出。并且Animal被编译成了一个立即执行函数，这里就产生了副作用。不过目前的babel已经足够先进，帮我们添加了之前提到了`#__PURE__`注释。

所以最好在打包的时候，将bundle阶段（tree-shake阶段），放在当前被转换的代码还是 ESM 的阶段，然后把polyfill等其它转换操作放在之后。

## 小结

Tree-shaking已经被现代的rollup/webpack等其它bundler所默认支持，其核心就是 ESM 的静态`import` / `export`。
在大多数情况下，开发者在编写代码时只需要注意编写规范，就已经能够得到良好的tree-shaking能力了。
