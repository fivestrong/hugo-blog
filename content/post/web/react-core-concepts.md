---
title: "React Core Concepts"
date: 2021-02-01T17:34:47+08:00
tags: ["react"]
categories: ["web"]
draft: true
---

## React components 简介

React每一个组件都可以作为用户界面上的一个独立单元，所见即所得，你在组件里面写啥浏览器就会呈现啥效果。为了能够运行不报错，每个组件必须返回至少一个UI元素。

创建React组件的方式有两种，通过JavaScript `class` 和 `function`语法来实现。下面通过 `class` 来定义组件

```react
import React from 'react';

class MainComponent extends React.Component {
  render(){
    return <h1> Hello, World!</h1>
  }
}
```

`class` 组件必须总是包含 `render()` 函数并且有`return`否则会报错。

下面是如何通过定义一个`function component`

```react
import React from 'react';

function MainComponent(){
  return <h1>Hello World</h1>
}
```

跟`class component`一样，`function component`必须通过`return返回`，但是你不必给它创建`render()`函数。

如果你的组件不想返回任何东西，可以返回 `null` 或者 `false`

```react
import React from 'react';

function MainComponent(){
  return null
}

class MainComponent extends React.Component {
  render(){
    return false
  }
}
```

因为`React`本质上就是一个`JavaScript`库，所有的组件代码可以保存在`.js`为扩展的文件中。你可能在其他开源项目中见到过`.jsx`扩展，大多数编译器对这两种扩展文件同等看待，所以用`.js`也挺好。

## 返回多个元素

默认情况下一个组件必须总是返回单一元素。当你想要返回多个元素，这时候就需要将这些元素包裹起来，比如使用`<div>`

```react
import React from 'react';

class MainComponent extends React.Component {
  render(){
    return (
    	<div>
      	<h1>Hello World!</h1>
        <h2>Learning to code with React</h2>
      </div>
    )
  }
}
```

这样会使得前端多渲染一层额外的`<div>`标签，如果你有强迫症的话，你可以使用空`tag`

```react
import React from 'react';

class MainComponent extends React.Component {
  render(){
    return (
    	<>
      	<h1>Hello World!</h1>
        <h2>Learning to code with React</h2>
      <>
    )
  }
}
```

这样就不会在浏览器端渲染额外的标签了，这个方法在函数组件中同样适用。

### 渲染到页面

组件创建了之后并不能直接渲染到屏幕上，还需要使用`ReactDOM.render()`函数，它接收两个参数：

1. 你想要渲染的组件
2. 渲染需要的HTML元素

也就是说你需要在React项目中创建一个HTML文件，否则React无处渲染组件。通常来说，一个最基本的HTML文件仅需要一个`div`就够了

```html
<div id="root"></div>
```

接下来将组件渲染到这个`div`中，这个操作需要从`react-dom`包中导入`ReactDOM`

```react
import React from 'react';
import ReactDOM from "react-dom";

class MainComponent extends React.Component {
  render(){
    return <h1>Hello World！</h1>
  }
}

ReactDOM.render(
	<MainComponent />,
  document.querySelector("#root")
)
```

演示代码 [a render in Code Sandbox](https://codesandbox.io/s/rendering-react-component-95rbl)

### 注释书写

在React组件中写注释跟你在传统的`JavaScript`代码中写的方式没有差别，你可以使用`//`来注释任何代码

```react
class MainComponent extends React.Component {
  render(){
    // const name = "xiaoming"
    // const age = 18
    return <h1>Hello World！</h1>
  }
}
```

同样可以使用`/* */`

```react
class MainComponent extends React.Component {
  render(){
    /*
    const name = "xiaoming"
    const age = 18
    */
    return <h1>Hello World！</h1>
  }
}
```

通常来说`/* */`注释方式用来注释`license`或者开发文档

```react
/** @license React v17.0.1
 * react.development.js
 *
 * Copyright (c) Facebook, Inc. and its affiliates.
 *
 * This source code is licensed under the MIT license found in the
 * LICENSE file in the root directory of this source tree.
 */
```

不过这个没有硬性规定，所以可以根据喜好使用。

### 第一个实例

