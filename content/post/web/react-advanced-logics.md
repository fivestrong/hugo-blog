---
title: "React Advanced Logics"
date: 2021-02-04T13:00:03+08:00
lastmod: 2021-02-04T13:00:03+08:00
draft: true
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

### useLocations hook

### useHistory hook

### useRouteMatch hook

