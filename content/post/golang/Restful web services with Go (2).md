---
title: "Restful Web Services With Go (2)"
date: 2020-03-05T09:54:01+08:00
tags: ["restful"]
categories: ["golang"]
draft: false
---

### 配置开发环境

Go 环境配置很简单，只需要安装Go编译器，设置好GOROOT 以及 GOPATH 环境变量。

网上有很多安装配置教程，这里就不多赘述了。

**理解GOPATH**

GOPATH用于指定golang的工作目录，告诉编译器你的源代码，二制进文件以及第三方包的存放位置。

Python程序员可能对 Virtualenv 这个工具比较熟悉，它可以同时创建多个项目（每个项目拥有不同版本的python解释器），当你想要编辑某个项目，只需要激活当前项目对应的python环境即可。

同样的道理，你也可以创建任意多个Go工作目录，当开发的时候，只需要把GOPATH设定到你的工作目录即可。

假设在家目录下创建一个目录，设置GOPATH到这里：

```shell
mkdir /home/user/workspace
export GOPATH=/home/user/workspace
```

如果我们这是时候安装一个第三方包

```shell
go get -u -v github.com/gorilla/mux // -u 安装并更新包的依赖  -v 查看安装具体信息
```

Go会把项目mux从github复制一份到workspace目录下面

GOPATH下面基本的目录结构有：

1. bin: 存储项目的二进制文件，这里编译好的二进制可以直接运行。
2. pkg: 包含项目的包信息，提供需要编译程序所需包的各种方法。
3. src: 存放用户源代码。

### 练手项目

**需求**：

创建一个REST服务，从OS网站镜像列表中选取最快的下载链接返回给用户。以Debian系统为例子，网址在：https://www.debian.org/mirror/list

**设计:**

API返回结果是最快镜像的URL，API设计文档如下：

| HTTP Verb | PATH            | Action | Resource    |
| --------- | --------------- | ------ | ----------- |
| GET       | /fastest-mirror | fetch  | URL: string |

**实现：**

1. 假设GOPATH在/home/user/workspace，项目目录为mirrorFinder，git-user为你自己的github用户名:

   ```shell
   mkdir -p $GOPATH/src/github.com/git-user/mirrorFinder
   ```

2. 创建主文件main.go:

   ```shell
   touch $GOPATH/src/github.com/git-user/mirrorFinder/main.go
   ```

3. 创建一个目录存放镜像数据，以及data.go文件

   ```shell
   mkdir $GOPATH/src/github.com/git-user/mirrors
   touch $GOPATH/src/github.com/git-user/mirrors/data.go
   ```

4. data.go中存放镜像列表

   ```go
   package mirrors
   
   // MirrorList is list of Debian mirror sites
   var MirrorList = []string{
   	"http://ftp.am.debian.org/debian/", "http://ftp.au.debian.org/debian/",
   	"http://ftp.at.debian.org/debian/", "http://ftp.by.debian.org/debian/",
   	"http://ftp.be.debian.org/debian/", "http://ftp.br.debian.org/debian/",
   	"http://ftp.bg.debian.org/debian/", "http://ftp.ca.debian.org/debian/",
   	"http://ftp.cl.debian.org/debian/", "http://ftp2.cn.debian.org/debian/",
   	"http://ftp.cn.debian.org/debian/", "http://ftp.hr.debian.org/debian/",
   	"http://ftp.cz.debian.org/debian/", "http://ftp.dk.debian.org/debian/",
   	"http://ftp.sv.debian.org/debian/", "http://ftp.ee.debian.org/debian/",
   	"http://ftp.fr.debian.org/debian/", "http://ftp2.de.debian.org/debian/",
   	"http://ftp.de.debian.org/debian/", "http://ftp.gr.debian.org/debian/",
   	"http://ftp.hk.debian.org/debian/", "http://ftp.hu.debian.org/debian/",
   	"http://ftp.is.debian.org/debian/", "http://ftp.it.debian.org/debian/",
   	"http://ftp.jp.debian.org/debian/", "http://ftp.kr.debian.org/debian/",
   	"http://ftp.lt.debian.org/debian/", "http://ftp.mx.debian.org/debian/",
   	"http://ftp.md.debian.org/debian/", "http://ftp.nl.debian.org/debian/",
   	"http://ftp.nc.debian.org/debian/", "http://ftp.nz.debian.org/debian/",
   	"http://ftp.no.debian.org/debian/", "http://ftp.pl.debian.org/debian/",
   	"http://ftp.pt.debian.org/debian/", "http://ftp.ro.debian.org/debian/",
   	"http://ftp.ru.debian.org/debian/", "http://ftp.sg.debian.org/debian/",
   	"http://ftp.sk.debian.org/debian/", "http://ftp.si.debian.org/debian/",
   	"http://ftp.es.debian.org/debian/", "http://ftp.fi.debian.org/debian/",
   	"http://ftp.se.debian.org/debian/", "http://ftp.ch.debian.org/debian/",
   	"http://ftp.tw.debian.org/debian/", "http://ftp.tr.debian.org/debian/",
   	"http://ftp.uk.debian.org/debian/", "http://ftp.us.debian.org/debian/",
   }
   ```

5. 编辑main.go文件

   ```go
   package main
   
   import (
   	"encoding/json"
   	"fmt"
   	"log"
   	"net/http"
   	"time"
   
   	"github.com/git-user/mirrors"
   )
   
   type response struct {
   	FastestURL string        `json:"fastest_url"`
   	Latency    time.Duration `json:"latency"`
   }
   
   func main() {
   	http.HandleFunc("/fastest-mirror", func(w http.ResponseWriter, r *http.Request) {
   		response := findFastest(mirrors.MirrorList)
   		respJSON, _ := json.Marshal(response)
   		w.Header().Set("Content-Type", "application/json")
   		w.Write(respJSON)
   	})
   	port := ":8000"
   	server := &http.Server{
   		Addr:           port,
   		ReadTimeout:    10 * time.Second,
   		WriteTimeout:   10 * time.Second,
   		MaxHeaderBytes: 1 << 20,
   	}
   	fmt.Printf("Starting server on port %s\n", port)
   	log.Fatal(server.ListenAndServe())
   }
   ```

   main函数通过net/http包创建了一个视图函数并启动HTTP服务，结构体response有两个字段：

   - Fastest_url: 最快镜像的地址
   - Latency: 本地请求README网页所花费的时间

6. 接下来实现findFastest函数，用于从镜像列表获取最快镜像。这里用到了Go routines，当并发请求这些URL；而在函数里面还用到了两个channle，用于接收返回的信息。这里比较巧妙的设计就是，当两个channel一旦接收到数据，函数会立刻返回response响应，这样其他没有完成的Go routines会被取消，最后得到的结果就是响应最快的镜像地址。

   ```go
   func findFastest(urls []string) response {
   	urlChan := make(chan string)
   	latencyChan := make(chan time.Duration)
   
   	for _, url := range urls {
   		mirrorURL := url
   		go func() {
   			start := time.Now()
   			_, err := http.Get(mirrorURL + "/README")
   			latency := time.Now().Sub(start) / time.Millisecond
   			if err == nil {
   				urlChan <- mirrorURL
   				latencyChan <- latency
   			}
   		}()
   	}
   	return response{<-urlChan, <-latencyChan}
   }
   ```

7. 现在使用install命令来编译:

   ```go
   go install github.com/git-user/mirrorFinder
   ```

   这一步做了两件事：

   - 编译mirrors包将编译后的结果放到$GOPATH/pkg 目录
   - 将编译好的二进制放到$GOPATH/bin目录

8. 运行程序

   ```go
   $GOPATH/bin/mirrorFinder
   ```

   这时候这个API服务就启动并监听在http://localhost:8000这个端口上。

9. 利用curl工具请求这个API

   ```go
   curl -i -X GET "http://localhost:8000/fastest-mirror"
   ```

   响应结果为

   ```go
   HTTP/1.1 200 OK
   Content-Type: application/json
   Date: Thu, 05 Mar 2020 06:05:58 GMT
   Content-Length: 64
   
   {"fastest_url":"http://ftp.kr.debian.org/debian/","latency":174}
   ```



### 使用 Swagger UI 生成API说明文档

使用Swagger UI最方便的方式是直接利用Docker启动，在这之前还需要准备一个JSON格式的配置文件，比如我们的服务需要的配置，openapi.json :

```json
{
  "openapi": "3.0.0",
  "info": {
    "title": "Mirror Finder Service",
    "description": "API service for finding the fastest mirror from the list of given mirror sites",
    "version": "0.1.1"
  },
  "servers": [
    {
      "url": "http://localhost:8000",
      "description": "Development server[Staging/Production are different from this]"
    }
  ],
  "paths": {
    "/fastest-mirror": {
      "get": {
        "summary": "Returns a fastest mirror site.",
        "description": "This call returns data of fastest reachable mirror site",
        "responses": {
          "200": {
            "description": "A JSON object of details",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "fastest_mirror": {
                      "type": "string"
                    },
                    "latency": {
                      "type": "integer"
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

该配置文件三个部分的作用：

- info: API服务相关的描述
- servers: 配置API服务所在的地址
- paths: 这部分包含服务器提供的所有API接口描述，包括请求体，响应类型，内容结构等。

开始通过Docker安装并使用Swagger UI :

1. 安装命令：

   ```shell
   docker pull swaggerapi/swagger-ui
   ```

2. 通过刚才的配置文件启动镜像

   ```shell
   docker run --rm -p 80:8080 -e SWAGGER_JSON=/app/openapi.json -v $GOPATH/src/github.com/git-user/:/app swaggerapi/swagger-ui
   ```

通过http://127.0.0.1就可以访问这个API文档了