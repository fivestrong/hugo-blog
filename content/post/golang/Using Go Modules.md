---
title: "Using Go Modules"
date: 2020-03-05T15:51:39+08:00
tags: ["go"]
categories: ["golang"]
draft: true
---

Go从1.11开始支持modules模式，如果当前目录不在*$GOPATH/src*目录下面并且当前目录或者父目录包含go.mod，Go命令行启用modules模式；如果是在*$GOPATH/src*路径，即便发现有go.mod文件Go命令仍然使用旧的GOPATH模式。

但是从1.13版本开始，在所有的开发环境下modele设置为默认模式。

下面记录一些Go modules 简单的使用方法

### 创建新的module

在任何位置创建一个空的目录，新建源文件hello.go

```go
package hello

func Hello() string {
  return "Hello, world."
}
```

创建测试文件hello_test.go

```go
package hello

import "testing"

func TestHello(t *testing.T) {
  want := "Hello, world."
  if got := Hello(); got != want {
    t.Errorf("Hello() = %q, want %q", got, want)
  }
}
```

因为没有go.mod文件，所以当前目录包含一个包，却不是一个模块。这时我们进入/home/gopher/hello并执行go test

显示结果为：

```go
$ go test
PASS
ok      _/home/gopher/hello    0.020s
```

最后一行是测试结果汇总。因为我们不在$GOPATH也不再模块内，Go命令无法确定当前目录的导入路径，所以它基于当前目录名称创建了一个假的：_/home/gopher/hello

接下来以当前目录作为模块根目录，使用 *go mod init* 创建go.mod文件

```go
$ go mod init github.com/hello
go: creating new go.mod: module example.com/hello
$ go test
PASS
ok      example.com/hello    0.020s
$
```

*go mod init* 写入文件的内容为:

```go
module github.gom/hello

go 1.14
```

go.mod 文件仅仅存在于模块的根目录下，如果子目录作为一个包需要被导入，直接以当前模块目录加包目录的形式导入就可以。比如你在hello目录下面创建了一个world包，它的导入路径就是github.com/hello/world。

### 添加依赖

Go modules 存在的意义就是为了提高对依赖包的管理

我们向hello.go导入 rsc.io/quote作为演示

```go
package hello

import "rsc.io/quote"

func Hello() string {
  return quote.Hello()
}
```

再次运行test

```go
# go test
go: finding module for package rsc.io/quote
go: found rsc.io/quote in rsc.io/quote v1.5.2
go: downloading golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
go: updating go.mod: existing contents have changed since last read
--- FAIL: TestHello (0.00s)
    hello_test.go:8: Hello() = "你好，世界。", want "Hello, world."
FAIL
exit status 1
FAIL	github.gom/hello	0.005s
```

go 命令通过go.mod中列出的模块的相关版本来进行导入调用。当遇到go.mod中没有出现过的模块，go 命令会自动查找下载相关模块最新版本并添加到go.mod中。go.mod目录仅记录直接依赖模块，如果第三方模块还有其他的依赖，会下载但不回记录。此时go.mod文件变成：

```go
module github.gom/hello

go 1.14

require rsc.io/quote v1.5.2
```

需要注意的是下载的第三方模块不存在本地，而是 $GOPATH/pkg/mod 目录下面。

下载直接依赖会引入其他依赖，通过 *go list -m all* 可以查看当前模块以及它的所有依赖

```go
github.gom/hello
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
rsc.io/quote v1.5.2
rsc.io/sampler v1.3.0
```

除了 go.mod文件，go 命令还维护一个文件 go.sum，这个文件包含特定模块版本的哈希加密值

```go
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c h1:qgOY6WgZOaTkIIMiVjBQcw93ERBE4m30iBm00nkL0i8=
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c/go.mod h1:NqM8EUOU14njkJ3fqMW+pc6Ldnwhi/IjpwHt7yyuwOQ=
rsc.io/quote v1.5.2 h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3Y=
rsc.io/quote v1.5.2/go.mod h1:LzX7hefJvL54yjefDEDHNONDjII0t9xZLPXsUe+TKr0=
rsc.io/sampler v1.3.0 h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/QiW4=
rsc.io/sampler v1.3.0/go.mod h1:T1hPZKmBbMNahiBKFy5HrXp6adAjACjK9JXDnKaTXpA=
```

go.sum 存在是为了确保将来再次下载这些模块的时候能和第一次下载的相同，确保模块没有被意外更改或者破坏。

### 更新依赖

Go  modules 版本号以语义标签的形式表示。假设版本号v0.1.2

- 0：表示主要版本
- 1：表示次要版本
- 2：表示补丁版本