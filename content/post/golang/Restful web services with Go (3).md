---
title: "Restful Web Services With Go (3)"
date: 2020-03-16T15:03:35+08:00
tags: ["restful"]
categories: ["golang"]
draft: true
---

学习目标：

- 理解Go自带的 `net/http` 包
- Go基本路由-ServeMux
- 轻量级 HTTP 路由 - httprouter
- 重量级 HTTP 路由 - gorilla/mux
- 构建一个短链接服务API

## 理解Go net/http 包

一个Web服务的基本功能就是响应HTTP请求，Go语言自带了一个 `net/http` 包，可以用来创建HTTP服务以及客户端。

可以通过一个例子来了解一下这个包的使用，创建的这个服务是接受请求，返回服务器时间。

1. 在你的工作目录下新建项目目录，并使用 go module 初始化

   ```shell
   mkdir healthCheck
   cd healthCheck
   go mod init healthCheck
   touch main.go
   ```

2. 编辑 main.go 文件，导入包 net/http 创建路由函数 HealthCheck，其中 `http.HandleFunc` 接收一个路由以及路由方法，返回 `http.ResponseWriter`。 ListenAndServe 创建一个HTTP服务，如果创建失败就报错；第一参数是 address:port 格式的监听地址端口，第二参数是使用的路由引擎，这里传入 nil 表示使用默认的。

   ```go
   package main
   
   import (
   	"io"
   	"log"
   	"net/http"
   	"time"
   )
   
   
   // HealthCheck API returns date time to client
   func HealthCheck(w http.ResponseWriter, req *http.Request){
   	currentTime := time.Now()
   	io.WriteString(w, currentTime.String())
   }
   
   func main(){
   	http.HandleFunc("/health", HealthCheck)
   	log.Fatal(http.ListenAndServe(":8000", nil))
   }
   ```

3. 启动服务

   ```go
   go run main.go
   ```

4. 打开另外一个 shell 或者 浏览器，访问创建的路由。这里使用 curl 请求

   ```go
   ➜  ~ curl -X GET http://localhost:8000/health
   2020-03-16 16:02:43.661315 +0800 CST m=+97.612028469
   ```

Go 有多种方法处理请求以及响应，这里使用的是 io 包写入响应。在Web开发中，我们更多的是使用模板来生成响应。Go默认的URL handlers 使用的是 ServeMux，下面来更详细的认识一下。

## Go 基础路由管理器 ServeMux

ServeMux 是一个HTTP请求的多路复用器。它使用 ServeHTTP 函数来处理分离路由的逻辑，所以如果我们在一个 Go 结构体中实现了 ServeHTTP 方法，我们也可以自己创建一个路由的多路复用器。

路由与响应函数的关系类似于 Go 中的字典(map)，路由作为 key ，函数作为 value。每次请求，Go都会查找这个字典，找到匹配的路由，执行相应的 ServeHTTP函数。接下来我们以一个生成 UUID 的API作为例子。

### 使用 ServeMux 开发一个 生成UUID的API

UUID是资源或事务的唯一标识符，在 HTTP 请求认证中经常使用到。

1. 在你的工作目录下新建项目目录，并使用 go module 初始化

   ```shell
   mkdir uuidGenerator
   cd uuidGenerator
   go mod init uuidGenerato
   创建如下结构文件
   ├── go.mod
   ├── main.go
   └── uuid
       └── uuidGenerator.go
   ```

2. 任何实现了特定的HTTP方法的结构体都可以成为 ServeMux。比如，我们创建一个 UUID struct 并实现 ServeHTTP 方法，以下代码在 uuidGenerator.go 中

   ```go
   package uuidGenerator
   
   import (
   	"crypto/rand"
   	"fmt"
   	"net/http"
   )
   
   // UUID is a custom multiplexer
   type UUID struct {
   
   }
   
   func (p *UUID) ServeHTTP(w http.ResponseWriter, r *http.Request){
   	if r.URL.Path == "/" {
   		giveRandomUUID(w, r)
   		return 
   	}
   	http.NotFound(w, r)
   	return 
   }
   
   func giveRandomUUID(w http.ResponseWriter, r *http.Request) {
   	c := 10
   	b := make([]byte, c)
   	_, err := rand.Read(b)
   	if err != nil{
   		panic(err)
   	}
   	fmt.Fprintf(w, fmt.Sprintf("%x", b))
   }
   ```

   这里的 UUID 结构体的作用类似于 ServeMux。我们可以通过获取不同的请求的URL路径，来指定相应的响应函数。这里的 giveRandomUUID 就是这样的函数。而 crypto 包中 Read 函数可以向一个 byte array 中随机的填入数字。

3. 接下来在主函数中使用我们创建的这个结构体，将它传入到 http.ListenAndServe 方法中。

   ```go
   package main
   
   import "net/http"
   import u "uuidgenerator/uuid"
   
   func main(){
   	mux := &u.UUID{}
   	http.ListenAndServe(":8000", mux)
   }
   ```

4. 启动服务

   ```go
   go run main.go
   ```

5. 使用 curl 测试

   ```go
   ➜  ~ curl -X GET http://localhost:8000/
   26ff6499344867139997
   ```

### 使用 ServeMux 添加多个路由

假设我们现在需要一个API用来随机生成不同类型的数字，比如整型、浮点型等等。当有多段路由需要匹配对应响应函数，这时候自定义mux就需要通过多个 if/else 判断条件来检测不同 URL 。为了简化操作，我们使用Go内置的 ServeMux 来处理多段路由，内置ServeMux的使用代码大致如下：

```go
newMux := http.NewServeMux()
newMux.HandleFunc("/randomFloat", func(w http.ResponseWriter, r *http.Request){
  fmt.Fprintln(w, rand.Float64())
})

newMux.HandleFunc("/randomInt", func(w, http.ResponseWriter, r *http.Request){
  fmt.Fprintln(w, rand.Int(100))
})
```

在项目目录下新建 main.go 完整的代码如下:

```go
package main

import (
	"fmt"
	"math/rand"
	"net/http"
)

func main() {
	newMux := http.NewServeMux()
	newMux.HandleFunc("/randomFloat", func(w http.ResponseWriter, r *http.Request){
  fmt.Fprintln(w, rand.Float64())
})

	newMux.HandleFunc("/randomInt", func(w http.ResponseWriter, r *http.Request){
  fmt.Fprintln(w, rand.Intn(100))
})
	http.ListenAndServe(":8000", newMux)
}
```

测试结果：

```go
go run main.go 
➜  ~ curl -X GET http://127.0.0.1:8000/randomFloat
0.6046602879796196
➜  ~ curl -X GET http://127.0.0.1:8000/randomInt
87
```

这些是Go提供的基本路由管理功能，接下来介绍一些更加流行并且用途比较广泛的路由管理框架。

## 轻量级路由管理 Httprouter

`httprouter` 是一个可以通过API优雅管理简单路由的框架。如果使用过Django框架的用户，`httprouter` 提供相似的功能

- 允许路径变量
- 适配REST方法（GET,POST,PUT等等）
- 不影响性能

`httprouter` 的其他优势：

- 与内置的 `http.Handler` 兼容
- 明确了请求的匹配方式，匹配到一条，或者一条也匹配不到
- 路由的设计鼓励构建合理的、分层的 RESTful APIs
- 可以构建简单高效的静态文件系统

### 简单使用 httprouter

1. 通过 go get 安装 `httprouter` 

   ```go
   go get github.com/julienschmidt/httprouter
   
   导入
   import "github.com/julienschmidt/httprouter"
   ```

2. 简单案例：获取Go编译器版本；获取给定文件内容

3. 创建项目

   ```go
   touch httprouterExample/main.go
   go mod init httproute
   ```

   预先定义的路由是：

   - /api/v1/go-version
   - /api/v1/show-file/:name   // :name 路径参数，Go默认路由没有这个功能

   ```go
   package main
   
   import (
   	"fmt"
   	"log"
   	"io"
   	"net/http"
   	"os/exec"
   
   	"github.com/julienschmidt/httprouter"
   )
   
   
   func getCommandOutput(command string, arguments ...string) string {
   	out, _ := exec.Command(command, arguments...).Output()
   	return string(out)
   }
   
   func goVersion (w http.ResponseWriter, r *http.Request, params httprouter.Params) {
   	response := getCommandOutput("/usr/local/bin/go", "version")
   	io.WriteString(w, response)
   	return 
   }
   
   func getFileContent(w http.ResponseWriter, r *http.Request, params httprouter.Params){
   	fmt.Fprintln(w, getCommandOutput("/bin/cat", params.ByName("name")))
   }
   
   func main() {
   	router := httprouter.New()
   	router.GET("/api/v1/go-version", goVersion)
   	router.GET("/api/v1/show-file/:name", getFileContent)
   
   	log.Fatal(http.ListenAndServe(":8000", router))
   }
   ```

   可以看到代码中用到了 `exec.Command` 来执行系统命令，Output 方法返回执行结果。在两个路由函数中都用到了这个功能。

   在相同的目录下创建两个测试用的文本文件

   Greek.txt

   ```reStructuredText
   Οἱ δὲ Φοίνιϰες οὗτοι οἱ σὺν Κάδμῳ ἀπιϰόμενοι.. ἐσήγαγον διδασϰάλια ἐς τοὺς ῞Ελληνας ϰαὶ δὴ ϰαὶ γράμματα, οὐϰ ἐόντα πρὶν ῞Ελλησι ὡς ἐμοὶ δοϰέειν, πρῶτα μὲν τοῖσι ϰαὶ ἅπαντες χρέωνται Φοίνιϰες· μετὰ δὲ χρόνου προβαίνοντος ἅμα τῇ ϕωνῇ μετέβαλον ϰαὶ τὸν ϱυϑμὸν τῶν γραμμάτων. Περιοίϰεον δέ σϕεας τὰ πολλὰ τῶν χώρων τοῦτον τὸν χρόνον ῾Ελλήνων ῎Ιωνες· οἳ παραλαβόντες διδαχῇ παρὰ τῶν Φοινίϰων τὰ γράμματα, μεταρρυϑμίσαντές σϕεων ὀλίγα ἐχρέωντο, χρεώμενοι δὲ ἐϕάτισαν, ὥσπερ ϰαὶ τὸ δίϰαιον ἔϕερε ἐσαγαγόντων Φοινίϰων ἐς τὴν ῾Ελλάδα, ϕοινιϰήια ϰεϰλῆσϑαι.
   ```

   Chinese.txt

   ```reStructuredText
   登鹳雀楼
   唐代：王之涣
   
   白日依山尽，黄河入海流。
   欲穷千里目，更上一层楼。
   ```

   运行程序

   ```go
   go run main.go
   ```

   通过浏览器或者curl来获取结果

   ```shell
   ➜  ~ curl -X GET http://127.0.0.1:8000/api/v1/go-version
   go version go1.14 darwin/amd64
   ➜  ~ curl -X GET http://127.0.0.1:8000/api/v1/show-file/greek.txt
   Οἱ δὲ Φοίνιϰες οὗτοι οἱ σὺν Κάδμῳ ἀπιϰόμενοι.. ἐσήγαγον διδασϰάλια ἐς τοὺς ῞Ελληνας ϰαὶ δὴ ϰαὶ γράμματα, οὐϰ ἐόντα πρὶν ῞Ελλησι ὡς ἐμοὶ δοϰέειν, πρῶτα μὲν τοῖσι ϰαὶ ἅπαντες χρέωνται Φοίνιϰες· μετὰ δὲ χρόνου προβαίνοντος ἅμα τῇ ϕωνῇ μετέβαλον ϰαὶ τὸν ϱυϑμὸν τῶν γραμμάτων. Περιοίϰεον δέ σϕεας τὰ πολλὰ τῶν χώρων τοῦτον τὸν χρόνον ῾Ελλήνων ῎Ιωνες· οἳ παραλαβόντες διδαχῇ παρὰ τῶν Φοινίϰων τὰ γράμματα, μεταρρυϑμίσαντές σϕεων ὀλίγα ἐχρέωντο, χρεώμενοι δὲ ἐϕάτισαν, ὥσπερ ϰαὶ τὸ δίϰαιον ἔϕερε ἐσαγαγόντων Φοινίϰων ἐς τὴν ῾Ελλάδα, ϕοινιϰήια ϰεϰλῆσϑαι.
   ➜  ~ curl -X GET http://127.0.0.1:8000/api/v1/show-file/chinese.txt
   登鹳雀楼
   唐代：王之涣
   
   白日依山尽，黄河入海流。
   欲穷千里目，更上一层楼。
   ```

   ### 构建一个简单的静态文件服务器

   通常，为了提供给客户端从服务器端获取静态文件的服务，我们需要借助Apache2或者Nginx。我们也可以使用 httprouter 自己实现一个。

    为了提供从Go server 获取文件的功能，我们需要定义一个特定的路由，比如

   ```go
   /static/*
   ```

   思路是使用 http 包提供的 Dir 方法加载文件，将获取到的信息返回给 httprouter。httprouter 还提供了一个好用的 ServeFiles 函数，能够映射获取到的文件。

   接下来创建一个 static 静态目录，位置根据自己的实际情况自定义.

   ```shell
   mkdir -p /users/git-user/static
   ```

   将上面的Greek.txt Chinese.txt 拷贝到这个目录下，作为示例使用。

   1. 创建项目目录，配置go mod

      ```shell
      mkdir fileServer
      touch fileServer/main.go
      go mod init fileserver
      ```

   2. 编写 main.go

      ```go
      package main
      
      import (
      	"log"
      	"net/http"
      
      	"github.com/julienschmidt/httprouter"
      )
      
      func main() {
      	router := httprouter.New()
      
      	// Mapping to methods is possible with HttpRouter
      	router.ServeFiles("/static/*filepath", http.Dir("/Users/git-user/static"))
      
      	log.Fatal(http.ListenAndServe(":8000", router))
      }
      ```

   3. 运行程序

      ```shell
      go run main.go
      ```

   4. 测试

      ```shell
      ➜  ~ curl -X GET http://127.0.0.1:8000/static/chinese.txt
      登鹳雀楼
      唐代：王之涣
      
      白日依山尽，黄河入海流。
      欲穷千里目，更上一层楼。
      ```

## 强大的路由管理 gorilla/mux

Mux 表示 多路复用器（multiplexer），gorilla/mux 的设计目标就是实现处理多个HTTP路由与不同 handlers 对应关系。Handlers 可以简单的理解为处理相应请求的对应函数。

Gorilla/mux 有以下特性：

- 路径匹配
- 查询参数匹配
- 域名匹配
- 二级域名匹配
- 反向URL生成

### 简单使用 gorilla/mux

1. 获取包

   ```shell
   go get -u github.com/gorilla/mux
   ```

2. 包导入使用方法

   ```go
   import "github.com/gorilla/mux"
   ```

gorilla/mux 处理 handler 的方式跟基本的 ServeMux 类似，但是不同于 httprouter ，它将HTTP所有的请求信息都封装到了一个请求对象中。

Gorilla/mux API 提供了三个重要的工具

- mux.NewRouter 方法
- *http.Request 对象
- *http.ResponseWriter 对象

NewRouter 方法创建一个新的 router 实例，用于映射路由和处理函数。gorilla/mux 向处理函数传递封装过的 *http.Request *http.ResponseWriter 对象，这些对象包含很多额外信息，比如 headers, path parmeters, request body, query parameters。接下来学习一下使用两种不同的风格来定义和使用路由。

#### 路径匹配

一个基于 GET 方法的路径匹配的 URL 如下:

```shell
https://example.org/articles/books/123
```

这里的基础路径为 https://example.org/articles/ 其中的 books 123 都是路径参数，下面的例子来介绍怎么获取以及使用这些参数。

1. 创建项目，初始化 go mod 

   ```shell
   mkdir muxRouter
   touch muxRouter/main.go
   go mod init muxrouter
   ```

2. 代码实现 main.go

   ```go
   package main
   
   import (
   	"fmt"
   	"log"
   	"net/http"
   	"time"
   
   	"github.com/gorilla/mux"
   )
   
   
   func ArticleHandler(w http.ResponseWriter, r *http.Request){
   	vars := mux.Vars(r)
   	w.WriteHeader(http.StatusOK)
   	fmt.Fprintf(w, "Category is: %v\n", vars["category"])
   	fmt.Fprintf(w, "ID is: %v\n", vars["id"])
   }
   
   func main() {
   	r := mux.NewRouter()
   
   	r.HandleFunc("/articles/{category}/{id:[0-9]+}", ArticleHandler)
   	srv := &http.Server{
   		Handler: r,
   		Addr: "127.0.0.1:8000",
   		WriteTimeout: 15 * time.Second,
   		ReadTimeout: 15 * time.Second,
   	}
   	log.Fatal(srv.ListenAndServe())
   }
   ```

3. 运行程序

   ```shell
   go run main.go
   ```

4. 测试

   ```shell
   ➜  ~ curl  http://127.0.0.1:8000/articles/books/123
   Category is: books
   ID is: 123
   ```

上面的例子展示了如何匹配以及解析路径参数，除了这种方法还有一种比较常用的方法是通过请求参数获取。

#### 参数匹配

在HTTP请求中，查询参数和请求URL一同发送，gorilla/mux 会获取并收集这些请求参数。类似URL为

```shell
http://127.0.0.1:8000/articles?id=123&category=books  // 所有的请求参数以 ？ 开始
```

接下来将上面的程序修改一下，换成通过请求参数匹配。

```go
// 在 main.go 文件中添加
r := mux.NewRouter()
r.HandleFunc("/articles", QueryHandler)
```

编写新的 QueryHandler 函数

```go
// QueryHandler handles the given query parameters
func QueryHandler(w http.ResponseWriter, r *http.Request){
	queryParams := r.URL.Query()
	w.WriteHeader(http.StatusOK)
	fmt.Fprintf(w, "Got parameter id:%s!\n", queryParams["id"][0])
	fmt.Fprintf(w, "Got parameter category:%s!", queryParams["category"][0])
}
```

运行并测试程序

```go
go run main.go

curl  http://127.0.0.1:8000/articles\?id\=1234\&category\=birds //在shell中需要转义符
Got parameter id:1234!
Got parameter category:birds!
```

### gorilla/mux 的其他特性

第一个就是它可以更具 `reverse mapping` 技术生成动态链接，这个链接是一个API资源的完整URL。这样生成的URL需要通过 Name 与路由联系起来，具体如下：

```go
r.HanderFunc("/articles/{category}/{id:[0-9]+}", ArticleHandler).Name("articleRoute")
```

这样我们就可以通过这个名字来动态生成链接了

```go
url, err := r.Get("articleRoute").URL("category", "books", "id", "123")
fmt.Printf(url.Path) //打印结果为 /articles/books/123
```

如果路由包含其他定义的参数，那么也需要将它传到 URL 方法中。

第二个特性就是 `路径前缀（path prefix）`, 它可以广泛的匹配所有可能的路径。一个典型的应用就是我们之前遇到过的静态文件服务。当我们从静态目录匹配文件，API路径应该匹配出到所在系统路径，并返回文件内容。

当我们吧 /static/ 定义为路径前缀，下面的路径应该都能匹配到：

```shell
http://localhost:8000/static/js/jquery.min.js
http://localhost:8000/static/index.html
http://localhost:8000/static/some_file.extension
```

用gorilla/mux 编写静态文件服务，需要用到的方法有 `PathPrefix` 和 `StripPefix`

```go
r.PathPrefix("/static/").Handler(http.StripPrefix("/static/", http.FileServer(http.Dir("/tmp/static"))))
```

第三个特性是 `严格的反斜线 strict slash`，激活这个特性可以让一个URL路径重定向到相同的以反斜线结尾的URL，反之亦然。

```go
r.StrictSlash(true)
r.Path("/articles/").Handler(ArticleHandler)
```

上面的例子将这个特性设置为 true，这样的话访问 /articles 也会被重定向，从而执行 ArticleHandler 响应函数；如果设置为 false ，那么 /articles/ 和 /articles 是不同路由路径

第四个特性是可以匹配经过解码的路径参数

```go
r.UseEncodedPath()
r.NewRoute().Path("/category/id")
```

这样匹配下面两个URL都是可行的

```shell
http://localhost:8000/books/2
http://localhost:8000/books%2F2
```



## 避免常见的SQL 注入

SQL注入是常见的攻击数据库的手段。如果一个请求没有做相应的防护，那么攻击者就可以在请求参数中拼接恶意字符串，这条请求直接被后端作为SQL 语句执行的话，就可能会造成潜在的安全事故。

比如说有一段代码，将用户名、密码插入到数据库中。它通过HTTP POST 请求来获取这些值来拼成SQL语句

```go
username := r.Form.Get("id")
password := r.Form.Get("category")
sql := "SELECT * FROM article WHERE id='" + username + "' AND category='" + password + "'"
Db.Exec(sql)
```

上面的代码直接使用获取到的值，如果这些值包含恶意的SQL 声明，比如说 -- 注释符， ORDER BY n

```shell
?category=books&id=10 ORDER BY 10--
```

如果程序直接返回数据库相应，这会泄露查询表的信息。然后攻击者通过不断更换参数尝试获取其他敏感信息。

```shell
Unknow column '10' in 'order clause'
```

避免这样的注入，可以使用以下预防措施：

- 在数据库中设置针对不同用户设置对表的权限等级
- 记录请求，及时发现异常
- 使用 `text/template` 包提供的 `HTMLEscapeString` 方法去过滤参数中的特殊字符
- 避免使用原生SQL语句，建议使用相应的ORM操作
- 避免将数据库 debug 信息返回给客户端
- 使用安全工具，比如 sqlmap ，去定期扫描程序，发现现在的威胁

## 实现一个URL转化短链接的API

这个服务的目标是将一个非常长的URL转换成简洁、清晰、容易记忆的短URL返回给用户。



 