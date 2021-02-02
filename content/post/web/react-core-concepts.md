---
title: "React Core Concepts"
date: 2021-02-02T21:42:38+08:00
lastmod: 2021-02-02T21:42:38+08:00
draft: true
description: ""
tags: ["react"]
categories: ["web"]
author: "Morgana"

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

官方提供了创建React项目的命令行工具，能够方便我们快速的开启项目。这个工具是`Create React App`

```shell
npx create-react-app myapp
```

创建项目之后进入并运行

```shell
cd myapp && npm start
```

这样就能看到官方的React模板试例，当前的项目目录是

```shell
.
├── README.md
├── node_modules
├── package.json
├── public
├── src
└── yarn.lock
```

其中`src/`目录存放React组件的源代码，打开`App.js`可以看到这个组件被`index.js`文件引用。

我们可以先不动别的，把`Edit <code>src/App.js</code> and save to reload.`这一行改为`Hello World!`，删除 `Learn React`这一行的a标签。

修改完之后我们可以从浏览器中立刻看到结果，就是我们修改之后的`Hello World!`

## JSX: React模板语言

通过前面我们了解到，一个组件必须返回一些内容：

```react
class MainComponent extends React.Component {
  render(){
    return <h1>Hello World！</h1>
  }
}
```

这里的标签`<h1>`看起来很像标准的HTML标签，但事实上它是React中特殊的模板语言，称为`JSX`。

`JSX`是`JavaScript`的扩展，它使得能够在React元素中使用JS，可以赋值给变量，可以作为函数的返回值，比如：

```react
class MainComponent extends React.Component {
  render(){
    const element = <h1>Hello World！</h1>
    return element
  }
}
```

JSX中可以加入样式class，但是需要使用额外的名称`className`

```react
const App = <h1 className='text-lowercase'>Hello World</h1>
```

JSX中的注释方式有些特别，你可能之前接触过HTML语言，它里面使用`<!-- -->`语法来注释，但是在React中这样写会报`Unexpected token`

```react
export default function App(){
  return (
  <div>
    <h1>Hello World</h1>
      	<!-- <p>My name is xiaoming</p> -->
     <p>Nice to meet you!</p>
   </div>
  )
}
```

正确的方式使使用`{/* comment */}`

```react
export default function App(){
  return (
  <div>
    <h1>Hello World</h1>
      {/* <p>My name is xiaoming</p> */}
     <p>Nice to meet you!</p>
   </div>
  )
}
```

多行注释

```react
export default function App(){
  return (
  <div>
    {/* <h1>Commenting in React and JSX</h1>
      <p>My name is xiaoming</p> 
      <p>Nice to meet you!</p> */}
   </div>
  )
}
```

小技巧：最新的IDE一般都会带有注释的快捷键，比如VSCode中使用`CTRL+/`  `command+/`(macOS)就可以自动实现注释

## JSX中使用JavaScript

JSX中使用`{}`来包含JavaScript语法：

```react
const lowercaseClass = 'text-lowercase'; 
const text = 'Hello World!'; 
const App = <h1 className={lowercaseClass}>{text}</h1>;
```

这便是React元素与HTML的不同之处，你在真正的HTML中没法通过`{}`来直接引用JavaScript。

这样做的好处就是，我们不需要再学习一套新的模板语法，而是直接使用JavaScript函数来控制模板的显示输出。

比如说我们有一个用户的数组需要显示:

```javascript
const users = [ 
  { id: 1, name: 'Xiaoming', role: 'Web Developer' }, 
  { id: 2, name: 'Xiaohong', role: 'Web Designer' }, 
  { id: 3, name: 'Xiaowang', role: 'Team Leader' }, 
]
```

我们可以使用`map()`函数来遍历数组

```react
import React from "react"

function App() {

const users = [ 
  { id: 1, name: 'Xiaoming', role: 'Web Developer' }, 
  { id: 2, name: 'Xiaohong', role: 'Web Designer' }, 
  { id: 3, name: 'Xiaowang', role: 'Team Leader' }, 
]

return (
    <>
    <p>The currently active users list:</p>
    <ul>
      {
        users.map(function(user){
          return ( <li> {user.name} as the {user.role} </li>)
        })
      }
    </ul>
    </>
  )
}
```

上面的例子会遍历`users`中的所有元素，生成对应的`<li>`，以HTML列表的形式显示在浏览器中。

还需要注意的一点是，上述方法还会报一个警告，说每一个子列需要`"key"`参数。这个key在React中有特殊的作用，它用于唯一操作列表中子元素的增删改查。

推荐使用你数据中的独一无二的标识来作为key值。比如上面的例子，`id`就可以作为这个唯一值:

```react
return (
  
<li key={user.id}>
    {user.name} as the {user.role}  
</li>
  
)
```

要是实在找不到独一无二的key，也可以使用数组的下标:

```react
{
users.map(function(user, index){
  return ( 
      <li key={index}>
        {user.name} as the {user.role} 
      </li> 
    ) 
})
}
```

但是最好不要这么做，可能会出现未知的BUG

## 多组件合并

