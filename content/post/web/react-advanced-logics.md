---
title: "React Advanced Logics"
date: 2021-02-04T13:00:03+08:00
lastmod: 2021-02-05T14:46:03+08:00
draft: false
description: ""
tags: ["react"]
categories: ["web"]
author: "Morgana"
---

## AJAX调用

基于现在流行的前后端分离架构，通过前端调用后端API的需求就很有必要。但是React本身只能使用自身state和props中的数据来渲染UI，这就需要你自己使用一个基于AJAX技术来获取数据的函数库了。

当前主流的库有：[Fetch](https://github.com/github/fetch)、[Axios](https://github.com/axios/axios)、[Superagent](https://github.com/visionmedia/superagent)，这些HTTP请求库区别不大，挑喜欢用。

React中请求AJAX的时机一般是组件第一次被挂载到浏览器DOM，这时候你就可以根据获取的数据来更行状态了。

类组件将AJAX请求放到`componentDidMount()`生命周期中，然后数据取得之后使用`setState`方法赋值给变量。下面是一个获取GitHub用户数据API的请求。

```react
export default class App extends React.Component {
  constructor(props){
    super(props)
    this.state = {
      data: []
    };
  }
  
  componentDidMount(){
    fetch("https://api.github.com/users?per_gage=3")
    .then((res) => res.json())
    .then(
    	(data) => {this.setState({data: data});},
      (error) => {console.log(error)}
    );
  }
  
  render(){
    const { data } = this.state;
    return (
    	<div>
      	<h1>React AJAX call</h1>
        <ul>{data.map((item) => (<li key={item.id}>{item.login}</li>))}</ul>
      </div>
    )
  }
}
```

当同样的请求使用函数组件获取时，要将请求放到`useEffect()`hook中，并且第二个参数设置为`[]`，这样hook只在第一次渲染时运行。

```react
export default function App(){
  const [data, setData] = useState([]);

  useEffect(()=>{
    fetch("https://api.github.com/users?per_gage=3")
    .then((res) => res.json())
    .then(
    	(data) => {setData(data);},
      (error) => {console.log(error)}
    );
  }, []);

  return (
    <div className="App">
      <h1>React AJAX call</h1>
      <ul>
        {data.map((item) => (<li key={item.id}>{item.login}</li>))}
      </ul>
    </div>
  )
}
```

如果你的网络情况不是太好的话，你可能后明显的感觉到页面渲染延迟，这时我们可以添加一个加载中的提示，告诉用户耐心等待。

```react
export default function App(){
  const [data, setData] = useState([]);
  const [isLoading, setLoading] = useState(true)
  const [error, setError] = useState(false)

  useEffect(()=>{
    setTimeout(()=>{
      fetch("https://api.githu.com/users?per_gage=3")
      .then((res) => res.json())
      .then(
        (data) => {
          setData(data);
          setLoading(false);
        },
        (error) => {
          setError(error);
          setLoading(false);
        }
      );
    }, 2000);
  }, []);

  if (error) {
    return <div>Fetch request error: {error.message}</div>;
  } else if (isLoading) {
    return <h1>Loading data...</h1>
  }else{
    return (
      <div className="App">
        <h1>React AJAX call</h1>
        <ul>
          {data.map((item) => (<li key={item.id}>{item.login}</li>))}
        </ul>
      </div>
    );
  }
}
```

我们通过`isLoading`来控制输出，当数据没获取完成，它的值是`true`，显示正在加载的动画；当数据获取完成它被设置为`false`，能够正确显示获取到的数据。为了演示效果，这里还在`useEffect`中添加超时函数，用来模拟网络阻塞的情况。最后代码中还加了对错误信息的处理。

## 使用Axios

`Axios`是JavaScript对于HTTP的请求库，它跟原生的`fetch`API相似，但是添加了更多有用的特性(来自官方文档)：

- Make [XMLHttpRequests](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest) from the browser
- Make [http](http://nodejs.org/api/http.html) requests from node.js
- Supports the [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) API
- Intercept request and response
- Transform request and response data
- Cancel requests
- Automatic transforms for JSON data
- Client side support for protecting against [XSRF](http://en.wikipedia.org/wiki/Cross-site_request_forgery)

通常对于JavaScript开发者来说，Axios更加通用，它不仅能够自动转化JSON响应到JavaScript数组或者对象，同时无论是服务器端(node.js)或浏览器端都能使用，而fetch在Node.js中不是原生支持。

**在React中使用axios**

使用axios需要额外安装包

```shell
npm install axios
```

剩下的跟使用`fetch`一样，把请求放在`componentDidMount()`或者`useEffect hook`当中

```react
componentDidMount(){
  axios.get("https://api.github.com/users?per_gage=3")
  .then(
    (response)=>{
      this.setState({
        data: response.data
      });
    }
  )
  .catch(error => {
    console.log(error)
  })
}
```

```react
useEffect(()=>{
  setTimeout(()=>{
    axios.get("https://api.github.com/users?per_gage=3")
    .then((respone) => {
      setData(respone.data);
    })
    .catch((error) => {
      console.log(error);
    });
  }, 2000);
}, []);
```

**Axios请求示例**

Axios 支持所有的HTTP请求方法，你可以在请求之前对axios进行设置

```react
axios({
  method: 'get',
  url: 'https://api.github.com/users'
})
.then(function (response){
  console.log(response)
});

// 或者

axios({
  method: 'post',
  url: 'https://api.github.com/users',
  data: {
    firstName: 'Fred',
    lastName: 'Flintstone'
  }
})
.then(function (response){
  console.log(response)
});
```

或者直接使用对应的方法

```javascript
axios.get("https://api.github.com/users")
axios.post("https://api.github.com/users", {
    firstName: 'Fred',
    lastName: 'Flintstone'
})
axios.put("https://api.github.com/users")
axios.patch("https://api.github.com/users")
axios.delete("https://api.github.com/users")
```

**同时执行多个请求**

你可以使用`Promise.all()`配合Axios来并行发出多个请求，等到响应返回之后在执行接下来的逻辑：

```javascript
function getUserAccount(){
  return axios.get('/user/123456')
}

function getUserPermissions(){
  return axios.post('/user/123456/permissions', {
    firstName: 'Fred',
    lastName: 'Flintstone'    
  })
}

Promise.all([getUserAccount(), getUserPermissions()])
	.then(function (results)){
		const acct = results[0];
		const perm = results[1];    
}
```

多个响应以请求的顺序返回成一个数组

## React Router

React router 是为了解决React应用路由问题的第三方库，它封装了浏览器`history API`使你的应用UI与浏览器URL同步。当你访问`/about`页面，React Router会确保About相关页面渲染到屏幕。作为在浏览器运行的单页面应用，React Router提供的功能能让你方便的在浏览器端切换，而不用每个URL都向服务器发送请求。 

React Router有两个包：给React使用的`react-router-dom`，给React Native使用的`react-router-native`，我们web应用只用前者就可以了。

```shell
npm install react-router-dom
```

React Router通常会用到三个组件：`BrowserRouter`，`Route`，`Link`

```react
import { BrowserRouter as Router, Route} from 'react-router-dom';

class RouterNavigationSample extends React.Component {
  render(){
    return (
      <Router>
        <>
          <NavigationComponent />
          <Route exact path="/" component={Home} />
          <Route path="/about" component={About} />
        </>
      </Router>
    )
  }
}
```

`BrowserRouter`导入别名`Router`，作为所有React组件的父组件，在最外层。它会拦截浏览器请求的URL并匹配到正确的`Route`组件。例如浏览器URL是`localhost:3000/about`，`Router`会查找路径为`/about`对应的组件，这里我们向路径`/about`的`Route`注册的组件是`About`。

在根路径上我们加入了`exact`参数，如果没有这个参数，任何path中包含`/`的路由都会被渲染成`Home`组件。

`Link`组件被用作导航，替换传统的`<a>`标签。传统的锚点标签会在每次点击的时候完全刷新，它不适用于React 应用。

```react
class NavigationComponent extends React.Component {
  render(){
    return (
      <>
        <ul>
          <li><Link to="/">Home</Link></li>
          <li><Link to="/about">About page</Link></li>
        </ul>
        <hr/>
      </>
    )
  }
}
```

**动态路由**

接着我们来学习另外两个路由组件：`Switch` 和 `Redirect`。

`Switch`组件会在找到第一个匹配路由之后渲染，并停止向下匹配。比如说有下面的例子

```react
import { Ruote } from 'react-router';

<Route path="/about" component={About} />
<Route path="/:user" component={User} />
<Route component={NoMatch} />
```

上面的代码中，`/about`会匹配到全部的三个路由，导致它们全部被渲染并且彼此紧邻。通过使用`Switch`组件，路由仅在匹配到`About`之后就停止。

```react
import { Switch, Ruote } from 'react-router';

<Switch>
  <Route path="/about" component={About} />
  <Route path="/:user" component={User} />
  <Route component={NoMatch} />
</Switch>;
```

这里组件的顺序特别重要，所以确保在声明参数路由、404路由之前先整理好所有静态路由。

`Redirect`组件用于重定向，`from`参数填写旧的链接，`to`参数写将要转向的路由。

```react
import { Redirect } from 'react-router';
<Redirect from='/old-match' to='/will-match' />;
```

**嵌套路由**

直接放案例代码

```react
import { BrowserRouter as Router, Link, Route} from 'react-router-dom';

const users = [
  {
    id: '1',
    name: 'Nathan',
    role: 'Web Developer',
  },
  {
    id: '2',
    name: 'Johnson',
    role: 'React Developer',
  },
  {
    id: '3',
    name: 'Alex',
    role: 'Python Developer',
  }
]

const Home = () => {
  return <div>This is the home page</div>
};

const About = () => {
  return <div>This is the about page</div>
};

const Users = () => {
  return (
    <>
      <ul>
        {
          users.map(({name, id}) => (<li key={id}><Link to={`/users/${id}`}>{name}</Link></li>))
        }
      </ul>
      <Route path='/users/:id' component={User} />
      <hr/>
    </>
  );
};

const User = (({match}) => {
  const user = users.find((user) => user.id === match.params.id);

  return (
    <div>
      Hello! I'm {user.name} and I'm a {user.role}
    </div>
  );
})

class NavigationComponent extends React.Component {
  render(){
    return (
      <>
        <ul>
          <li><Link to="/">Home</Link></li>
          <li><Link to="/about">About page</Link></li>
          <li><Link to="/users">Users page</Link></li>
        </ul>
        <hr/>
      </>
    )
  }
}

class RouterNavigationSample extends React.Component {
  render(){
    return (
      <Router>
        <>
          <NavigationComponent />
          <Route exact path="/" component={Home} />
          <Route path="/about" component={About} />
          <Route path="/users" component={Users} />
        </>
      </Router>
    )
  }
}

export default function App(){
  return <RouterNavigationSample />
}
```

这里我们加了些用户数据，想要实现的功能是：通过路由展示当前用户列表，用户列表中每个元素又能指向用户详情。

首先可以先写好对应的路由和跳转超链接，接着创建`Users`组件，它显示所有用户列表，同时也要用到`Link`和`Route`来创建跳转到详情页面的子路由。

最后在用户详情`User`组件中通过`/users/:id`传过来的参数渲染对应用户的详细信息。

这里有一些需要了解的知识点，每次组件被特定的路由渲染，该组件都会接收到来自React Router的props。有三个路由参数会被向下传递，它们分别是:

`match` ，`location`， `history`，而接收`/:id`的参数正是`match.params.id`，最后的id便是路由中的名称，比如你换成了`/:userId`，那相应的参数应该改为

`match.params.userId`

了解了route props之后，现在我们可以重构一下`Users`组件中的代码

```react
const Users = ({match}) => {
  return (
    <>
      <ul>
        {
          users.map(({name, id}) => (<li key={id}><Link to={`${match.url}/${id}`}>{name}</Link></li>))
        }
      </ul>
      <Route path={`${match.url}/:id`} component={User} />
      <hr/>
    </>
  );
};

```

这样就实现了动态路由

**向路由组件传递参数**

你可能会觉得向路由组件传递参数跟普通的组件传递方式一样：

```react
<Route path="/about" component={About} user="Jelly" />
```

那你就大错特错了，React Router没办法将写在`Route`中的props转发到`component`的props中，我们需要另辟它法。

React Router 提供了 `render`参数，它接收一个方法，当URL被匹配了之后会被调用。这里的props就能够被组件的props接收到:

```react
<Route 
  path="/about"
  render={props => <About {...props} admin="Blus" />}
  />


// the component
const About = props => {
  return <div>This is the about page {props.admin}</div>;
}
```

首先你将React Router听歌的`props`作为参数传递给自定义的组件，这时组件能够使用 `match`， `loation`， `history`这些参数。同时你也可以提供自定义的参数，上面的例子中`admin`就是这样。

## React Router hooks

随着hooks的加入，React Router的开发者也提供了自己的hooks，你可以在自己的React函数组件中任意使用。它将大大提高你编写导航组件的效率。

### useParams hook

`useParams hook`能够直接返回当前路由中的动态参数

例如，目前有一个`/post/:slug的URL，按照我们之前的处理方法：

```react
export default function App(){
  return (
  	<Router>
    	<div>
      	<nav>
        	<ul>
          	<li><Link to="/">Home</Link></li>
            <li><Link to="/post/hello-world">First Post</Link></li>
          </ul>
        </nav>
        <Switch>
        	<Route path="/post/:slug" component={Post} />
          <Route path="/"><Home /></Route>
        </Switch>
      </div>
    </Router>
  );
}
```

这里我们将`Post`组件传入到`/post/:slug`路由当中，这样我们就可以在组件中使用`match.params`来取得参数

```react
// Old way to fetch parameters
function Post({ match }) {
  let params = match.params;
  return (
  	<div>
    	In React Router v4, you get parameters from the props.
      Current parameter is <strong>{params.slug}</strong>
    </div>
  );
}
```

这样做可行，但是如果项目中动态路由数目剧增，管理起来会很不方便。你需要了解哪个路由中的组件需要使用到props，哪个不需要。同时这个由`Route`传递下来的`match`对象，需要你手动的依次传递到下层的其他DOM组件。

这时候`useParams hook`就应运而生了，它能够在不使用组件props的情况下获取到当前路由的参数。

```react
<Switch>
	<Route path="/post/:slug" component={Post} />
  <Route path="/users/:id/:hash">
    <Users />
  </Route>
  <Route>
    <Home />
  </Route>
</Switch>

function Users() {
  let params = useParams();
  return (
    <div>
    	In React Router v5, You can use hooks to get paramters.
      <br />
      Current id parameter is <strong>{params.id}</strong>
      <br />
      Current hash parameter is <strong>{params.hash}</strong>
    </div>
  )
}
```

如果这时候`Users`组件的子组件同样需要用到这些参数，你只需要接着使用`useParams()`就可以了。

### useLocations hook

在React Router v4版本中，跟获取参数一样，你需要使用组件props模式来取得`location`对象。

```react
<Route path="/post/:number" component={Post} />

function Post(props){
  return (
    <div>
    	In React Router v4, you get the location object from props.
      <br />
      Current pathname: <strong>{props.location.pathname}<strong>
    </div>
  );
}
```

v5.1以后，可以使用`useLocation hook`获取

```react
<Route path="/users/:id/:password" >
  <Users />
</Route>

// new way to fetch location with hooks
function Users(){
  let location = useLocation();
  return (
    <div>
    	In React Router v5, You can use hooks to get location object.
      <br />
      Current pathname: <strong>{location.pathname}<strong>
    </div>
  );
}
```

### useHistory hook

```react
<Route path="/post/:slug" component={Post}>

// Old way to fetch history
function Post(props){
    return (
      <div>
      	In React Router v4, you get the history object from props.
        <br />
        <button type="button" onClick={() => props.history.goBack()}>Go back</button>
      </div>
    );
  }
```

使用`useHistory hook`

```react
<Route path="/users/:id/:hash">
  <Users>
</Route>


function Users(){
    let history = useHistory();
    return (
      <div>
      	In React Router v5, You can use hooks to get history object.
        <br />
        <button type="button" onClick={() => history.goBack()}>Go back</button>
      </div>
    );
  }
```

使用这些hooks能够极大的简化代码，我们不再需要在组件里面传递`component`、`render`参数，仅需要在`<Route>`中传递URl path，并且把需要渲染的组件包裹在它之间就行。

### useRouteMatch hook

有时候为了获取 match 对象，我们需要使用`<Route>`组件

```react
<Route path="/post">
  <Post />
</Route>


function Post(){
    return (
      <Route 
        path="/post/:slug"
        render={({ match }) => {
          return (
          	<div>Your current path: <strong>{match.path}</strong></div>
          );
        }}
        />
    );
  }
```

v5版本使用hooks的实现

```react
<Route path="/users">
  <Users />
</Route>

function Users() {
  let match = useRouteMatch("/users/:id/:hash");
  return (
  	<div>
    	In React Router v5, You can use useRouteMatch to get match object.
      <br />
      Current match path: <strong>{match.path}</strong>
    </div>
  )
}
```



## Context API

通常来说在React中，你的数据需要从顶层组件一层一层传递到下层组件，即使这些数据只有最后一层组件才需要用到。这种自上而下的数据流有一个好处就是能够精准的定位数据传递过程中在哪里出现了问题。

不过这种方法确实有点儿死板，所以React提供了类似定义全局变量的功能，这就是`Context API`。

在一个组件中定义的数据作为`provider`生产者，而其他使用这个数据的组件作为`consumer`消费者。通过Context API共享的数据可以理解为React 组件树中的全局变量，它可以用来实现的功能有：共享当前认证用户、选择的主题或者语言。

跟 `useState hook`使用类似，可以通过`React.createContext()`来创建，并传递默认值

```react
import React from 'react';

// default to 'cn'
const LanguageContext = React.createContext('cn')
```

这样我们就得到一个可以使用的`LanguageContext`，它提供了`LanguageContext.Provider`和 `LanguageContext.Consumer`两个组件。

**生成context**

context中的数据通常是由state提供，一般当state改变，context相应的数据也会改变。你需要使用`Provider`组件将自定义组件包起来

```react
import React from 'react';

const LanguageContext = React.createContext('cn')

function App() {
  const language = 'en'
  
  return (
    <LanguageContext.Provider value={language}>
      <Hello />
    </LanguageContext.Provider>
  )
}
```

这样`<Hello>`组件以及它的子组件都能使用`LanguageContext`提供的值。

**方法组件使用context内容的方式**

方法组件可以使用提供的 `useContext` hook

```react
function Hello(){
  const language = useContext(LanguageContext)
  
  if (language === "en"){
    return <h1>Hello!</h1>
  }
  return <h1>你好！</h1>
}
```

我们根据上面的代码修改，使得context值能够被改变

```react
import React, { useState, useContext } from 'react';

const LanguageContext = React.createContext()

function App() {
  const [language, setLanguage] = useState("en");
  
  const changeLangeuage = () => {
    if (language === "en"){
      setLanguage("cn")
    } else {
      setLanguage("en")
    }
  }
  
  return (
    <LanguageContext.Provider value={language}>
      <Hello />
      <button onClick={changeLanguage}>Change Language</button>
    </LanguageContext.Provider>
  )
}

function Hello(){
  const language = useContext(LanguageContext)
  
  if (language === "en"){
    return <h1>Hello!</h1>
  }
  return <h1>你好！</h1>
}
```

**在类组件中使用context值**

在类组件中你可以使用`Context.Consumer`组件来获取值。这个组件需要一个子函数，它会将当前的值传递进去。

```react
class Hello extends React.Component {
  render() {
    return (
    	<LanguageContext.Consumer>
        {(language) => {
          if (language === "en"){
            return <h1>Hello!</h1>
          }
          return <h1>你好！</h1>
        }}
      </LanguageContext.Consumer>
    );
  }
}
```

这种组件中套函数的方法，看着有点儿反人类，所以React官方还提供了另外一种方法，使用`contextType`，将context对象赋值给`contextType`，可以使用`this.context`得到context的值。

```react
class Hello extends React.Component {
  static contextType = LanguageContext;
  render() {
    const language = this.context;
    if (language === "en"){
      return <h1>Hello!</h1>
    }
    return <h1>你好！</h1>
  }
}
```



由于全局变量的特性，你可以给多个组件提供相同的值。也就是说一个`provider`可以有多个`consumers`。



## Some Tips

**不要使用变量展开的方式传递props**

```react
function MainComponent() {
  const props = { firstName: "Jack", lastName: "Skeld"}
  return <Hello {...props} />
}
```

这种方式将变量一股脑的传递到子组件，如果参数很多或者获取的数据来自其他API，全部传递的话，子组件会得到很多无用的数据，不便于定位数据。所以还是推荐用到什么数据就传递什么数据到子组件。

**使用propTypes和defaultProps来对参数做限制**

```react
import React from "react";
import ReactDOM from "react-dom";

function App() {
  return <Greeting name="Nathan" />;
}
    
function Greeting(props){
  return (
  	<p>
    	Hello! I'm {props.name},
      a {props.name} years old {props.occupation}.
      Pleased to meet you!
    </p>
  );
}

ReactDOM.render(<App />, document.getElementById("root"));
```

上面的例子中`props.name`、`props.occupation`没有被传递，但是运行不会报错，React会忽略这两个参数，直接渲染剩下的数据。

我们可以使用第三方包`propTypes`来对参数进行限制

```shell
npm install --save prop-types
```



```react
import React from "react";
import ReactDOM from "react-dom";
import PropTypes from "prop-types";

function App() {
  return <Greeting name="Nathan" />;
}
    
function Greeting(props){
  return (
  	<p>
    	Hello! I'm {props.name},
      a {props.name} years old {props.occupation}.
      Pleased to meet you!
    </p>
  );
}

Greeting.propTypes = {
  // name must be a string and defined
  name: PropTypes.string.isRequired,
  // age must be a number and defined
  age: PropTypes.number.isRequired,
  // occupation must be a string and defined
  occupation: PropTypes.string.isRequired,
}

Greeting.defaultProps = {
  name: "Nathan",
  age: 27,
  occupation: "Software Developer"
}

ReactDOM.render(<App />, document.getElementById("root"));
```

上面的代码我们通过`propTypes`限制了传递参数的类型以及是否必须要求提供，同时也加入了默认值。如果不定义默认值，我们不向组件传递值或者传递值类型错误，就会在console抛出告警。加入了默认值之后，不传值的话它会使用你定义的默认值。

