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



