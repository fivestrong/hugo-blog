---
title: "React Function Components and Hooks"
date: 2021-02-03T21:36:46+08:00
lastmod: 2021-02-03T21:36:46+08:00
draft: false
description: ""
tags: ["react"]
categories: ["web"]
author: "Morgana"
---

在之间的版本中，`state`、`lifecycle functions`等这些功能只能在类组件中使用，这使得函数组件的功能和使用被大大的限制了。

但是自从引入`React Hooks`，函数组件地位大大提升，跟类组件并驾齐驱，甚至官方更推荐使用简洁的函数组件。所以除了之前介绍的类组件，我们更应该多了解函数组件的特性，因为在真实项目中它被使用的越来越多。

## 基于函数组件

在之前的文章中我们提到了两种组件定义的方法，分别使用`class` 和 `function` 语法：

```react
import React from 'react';

class MyClassComponent extends React.Component {
  render(){
    return <h1>Hello World</h1>
  }
}

function MyFunctionComponent(){
  return <h1>Hello World</h1>
}
```

函数组件不像类组件，继承了React的`Component`，所以它没办法得到`state`，也没法通过生命周期函数来在不同的阶段执行代码。

另外一个不同之处在于类组件将接收到的props存在`this.props`对象中，而函数组件需要通过第一个参数来接收props:

```react
function MyFunctionComponent(props){
  return <h1> Hello World</h1>
}
```

下面是一个类组件调用函数组件的例子：

```react
import React from 'react';

class MyClassComponent extends React.Component {
  render(){
    return <MyFunctionComponent name="John">
  }
}

function MyFunctionComponent(props){
  return <h1>Hello World, {props.name}</h1>
}
```

## useState hook

`React hooks`通俗讲就是React包提供的一系列函数，它可以给组件添加额外的功能。值得注意的是，Hooks只能在函数组件中使用，类组件没法用。

React提供了10多个hooks，不过一般在我们写组件的时候最常用的也就那么两三个。我常用到的hooks是`useState`和`useEffect`。

`useState`hook接收一个参数，用来做状态初始化；返回两个值：当前状态和更新状态的函数。

```react
import React, {useState} from 'react'

function UserComponent(){
  const [name, setName] = useState('John')
}
```

这里用到了ES6的`array destructuring syntax`，意思就是将函数useState返回的第一个值赋值给`name`，第二个值赋值给`setName`（注意它是一个函数），使用方法：

```react
import React, {useState} from 'react'

function UserComponent(){
  const [name, setName] = useState('John')
  
  if(name === "John"){
    setName("Luke")
  }
  
  return <h1>Hello World! My name is {name}</h1>
}
```

这里我们能够直接使用`name`变量，并且使用`setName`函数来改变它的值，功能类似与之前类组件中的全局`setState()`函数。

如果你需要定义多个states，可以多次调用`useState`hook来实现。这个hook接收所有合法的JavaScript数据类型，比如说字符串、数字、布尔值、数组、对象等等。

```react
import React, { useState } from 'react'

function UserComponent(){
  const [name, setName] = useState('Jack')
  const [age, setAge] = useState(10)
  const [isLegal, setLegal] = useState(false)
  const [friends, setFriends] = useState(["John", "Luke"])
  
  return <h1>Hello World! My name is {name}</h1>
}
```

总的来说，`useState`给函数组件提供定义内部状态的能力。

## useEffect hook

`useEffect hook`整合了`componentDidMount`, `componentDidUpdate`,`componentWillUnmount`这几个生命周函数的功能。它用于设定监听服务、从后端API获取数据、在组件被销毁前清除数据等操作。

我们可以先看看通常的功能在类组件中实现的方法：

```react
class Example extends React.Component {
  constructor(props){
    this.state = {
      name: 'Nacy',
    };
  }
  
  componentDidMount(){
    console.log(
    	`didMount triggered: Hello I'm ${this.state.name}`
    );
  }
  
  componentDidUpdate(){
    console.log(
    	`didUpdate triggered: Hello I'm ${this.state.name}`
    );
  }
  
  render(){
    return (
    	<div>
      	<p>{`Hello I'm ${this.state.name}`}</p>
        <button onClick={() => {this.setState({name: 'Niclo'})}}>
          Change me
        </button>
      </div>
    );
  }
}
```

我们之前也提过，`componentDidMount`只在组件被添加到DOM树的时候被执行一次，之后的渲染都不会执行到其中函数。为了每次渲染都执行，这时候就需要使用到`componentDidUpdate`函数。

使用`useEffect hook`相当于同时拥有了`componentDidMount`和`componentDidUpdate`的特性，因为`useEffect`每次渲染都会被执行。它接受两个参数：

- (必须)每次渲染执行的函数
- (可选)一个包含需要监听状态改变的数组，当没有状态变量发生变化的时候，`useEffect`会跳过执行。

将上面的例子重写之后:

```react
const Example = props => {
  const [name, setName] = useState('Nacy');
  
  useEffect(()=>{
    console.log(`Hello I'm ${name}`);
  })
  
  return (
    <div>
      <p>{`Hello I'm ${this.state.name}`}</p>
      <button onClick={() => {setName('Niclo')}}>
        Change me
      </button>
    </div>
    );
}
```

需要注意的是`useEffect`中的函数会在每次渲染后被执行。这对于有些场景来说是对性能有影响的，我们可能只需要它在name变量被改变之后再运行，所以可以用到第二个可选参数：

```react
useEffect(() => {
  console.log(`Hello I'm ${name}`);
}, [name]);
```

如果你有需要在组件移除DOM树之前执行的代码，实现类似`componentWillUnmount`功能，你需要在第一个参数传入的函数中通过 `return` 调用这些代码

```react
  useEffect(()=>{
    console.log(`useEffect function`);
    
    return () => { console.log("componentWillUnmount effect"); }
  }, [name]);
```

为了实现类似`componentDidMount`功能。只执行一次代码，你可以向第二个参数传递一个空数组.

```react
  useEffect(()=>{
    console.log(`useEffect function`);
  }, []);
```

空数组表明该effect没有任何需要监听改变的依赖，所以当它被挂载之后将不再执行。

## rules of hooks

### 只在最顶层调用Hooks

不要在循环、条件语句以及内嵌方法中调用hooks。如果你需要根据条件使用hooks，将条件写到hooks函数中

低情商：

```react
if (name !=== ''){
  useEffect(function logger(){
    console.log(name)
  });
}
```

高情商：

```react
useEffect(function logger(){
  if (name !== ''){
    console.log(name)
  }
});
```

这条规则能够确保Hooks在每次组件被渲染时候都能按照顺序执行，当你有多个`useState` `useEffect`调用的时候，这尤为重要。

### 只在函数组件中调用

不要在纯JavaScript 函数中调用hooks，它应该只出现在函数组件以及自定义hooks中。遵守这条规则能够让组件逻辑保持清晰，便于理解。

## 根据条件渲染不同结果

你可以在JSX中根据条件来渲染不同的输出，比如说我们需要根据用户状态来显示登录和登出按钮:

```react
import React from "react"

function App(props){
  const {users} = props
  
  if (user){
      return (
        <>
          <h1>Hello there!</h1>
          <button>Login</button>
        </>
  		)
  }
  return (
      <>
        <h1>Hello there!</h1>
        <button>Logout</button>
      </>
  )
}

export default App
```

我们不需要使用`else`语句，因为React在遇到`return`会立刻返回，不再执行接下的代码。

## 善用变量来部分渲染

假如我们有这样一个需求：需要动态的渲染UI中的部分内容。

```react
import React from "react"

function App(props){
  const {user} = props
  
  let button = <button>Login</button>
      
  if (user){
    button = <button>Logout</button>
  }
  
  return (
    <>
    	<h1>Hello there!</h1>
      {button}
    </>
  )
}

export default App
```

区别与之前写两个返回表达式，这里你只需要将需要改变的部分存入变量，减少了静态UI部分的书写。

## 使用&&操作符

当有需求：只有特定需求满足才渲染否则返回null，比如说只有当用户有新邮件时候才渲染动态消息：

```react
import React from "react"

function App(props){
  const newEmails = 2
  
  return (
    <>
    	<h1>Hello there!</h1>
      {newEmails >0 && 
       <h2>
         You have {newEmails} new emails in your inbox.
       </h2>
      }
    </>
  )
}

export default App
```

## 使用三元表达式

同样的三元表达式经常用来条件渲染组件

```react
import React from "react"

function App(props){
  const {user} = props
  
  return (
    <>
    	<h1>Hello there!</h1>
    	{ user? <button>Logout</button>:<button>Login</button> }
    </>
  )
}

export default App
```

