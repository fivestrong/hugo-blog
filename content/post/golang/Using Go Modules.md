---
title: "Using Go Modules"
date: 2020-03-05T15:51:39+08:00
tags: ["go"]
categories: ["golang"]
draft: false
---

Go从1.11开始支持modules模式，如果当前目录不在`$GOPATH/src`目录下面并且当前目录或者父目录包含go.mod，Go命令行启用modules模式；如果是在`$GOPATH/src`路径，即便发现有go.mod文件Go命令仍然使用旧的GOPATH模式。

但是从1.13版本开始，在所有的开发环境下module设置为默认模式。

下面记录一些`Go modules `简单的使用方法

### 创建新的module

在任何位置创建一个空的目录，新建源文件`hello.go`

```go
package hello

func Hello() string {
  return "Hello, world."
}
```

创建测试文件`hello_test.go`

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

因为没有go.mod文件，所以当前目录是一个包，不是模块。这时我们进入`/home/gopher/hello`并执行`go test`

显示结果为：

```go
$ go test
PASS
ok      _/home/gopher/hello    0.020s
```

最后一行是测试结果汇总。因为我们不在`$GOPATH`也不在模块内，Go命令无法确定当前目录的导入路径，所以它基于当前目录名称创建了一个假的：`_/home/gopher/hello`

接下来以当前目录作为模块根目录，使用 `go mod init` 创建`go.mod`文件

```go
$ go mod init github.com/hello
go: creating new go.mod: module example.com/hello
$ go test
PASS
ok      github.com/hello    0.020s
$
```

*go mod init* 写入文件的内容为:

```go
module github.com/hello

go 1.14
```

go.mod 文件仅仅存在于模块的根目录下，如果子目录作为一个包需要被导入，直接以当前模块目录加包目录的形式导入就可以。比如你在hello目录下面创建了一个world包，它的导入路径就是`github.com/hello/world`。

### 添加依赖

Go modules 存在的意义就是为了提高对依赖包的管理

我们向`hello.go`导入 `rsc.io/quote` 作为演示

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
FAIL	github.com/hello	0.005s
```

go 命令通过go.mod中列出的模块的相关版本来进行导入调用。当遇到go.mod中没有出现过的模块，go 命令会自动查找下载相关模块最新版本并添加到go.mod中。go.mod目录仅记录直接依赖模块，如果第三方模块还有其他的依赖，会下载但不回记录。此时go.mod文件变成：

```go
module github.com/hello

go 1.14

require rsc.io/quote v1.5.2
```

需要注意的是下载的第三方模块不存在本地，而是 `$GOPATH/pkg/mod `目录下面。

下载直接依赖会引入其他依赖，通过 `go list -m all` 可以查看当前模块以及它的所有依赖

```go
github.com/hello
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
rsc.io/quote v1.5.2
rsc.io/sampler v1.3.0
```

除了 `go.mod` 文件，go 命令还维护一个文件 `go.sum`，这个文件包含特定模块版本的哈希加密值

```go
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c h1:qgOY6WgZOaTkIIMiVjBQcw93ERBE4m30iBm00nkL0i8=
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c/go.mod h1:NqM8EUOU14njkJ3fqMW+pc6Ldnwhi/IjpwHt7yyuwOQ=
rsc.io/quote v1.5.2 h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3Y=
rsc.io/quote v1.5.2/go.mod h1:LzX7hefJvL54yjefDEDHNONDjII0t9xZLPXsUe+TKr0=
rsc.io/sampler v1.3.0 h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/QiW4=
rsc.io/sampler v1.3.0/go.mod h1:T1hPZKmBbMNahiBKFy5HrXp6adAjACjK9JXDnKaTXpA=
```

`go.sum` 存在是为了确保将来再次下载这些模块的时候能和第一次下载的相同，确保模块没有被意外更改或者破坏。

### 更新依赖

`Go  modules` 版本号以语义标签的形式表示。假设版本号v0.1.2

- 0：表示主要版本
- 1：表示次要版本
- 2：表示补丁版本

从之前的信息我们了解到text这个包版本号很低，我们现在将它更新到最新版本。

```shell
# go get golang.org/x/text
go: golang.org/x/text upgrade => v0.3.2
go: downloading golang.org/x/text v0.3.2
# go test
PASS
ok  	github.com/hello	0.005s
```

更新后测试一切正常，再看一下` go list -m all` 和 `go.mod `的输出

```shell
# go list -m all 
github.gom/hello
golang.org/x/text v0.3.2
golang.org/x/tools v0.0.0-20180917221912-90fa682c2a6e
rsc.io/quote v1.5.2
rsc.io/sampler v1.3.0
# cat go.mod
module github.com/hello

go 1.14

require (
	golang.org/x/text v0.3.2 // indirect
	rsc.io/quote v1.5.2
)
```

可见`go.mod`文件也被更新了，注释中的 `indirect` 表示text包没有直接被当前包引用，而是被其他包直接引用。

再更新一下` rsc.io/sampler` 试试

```shell
# go get rsc.io/sampler
go: rsc.io/sampler upgrade => v1.99.99
go: downloading rsc.io/sampler v1.99.99
# go test
--- FAIL: TestHello (0.00s)
    hello_test.go:8: Hello() = "99 bottles of beer on the wall, 99 bottles of beer, ...", want "你好，世界。"
FAIL
exit status 1
FAIL	github.com/hello	0.005s
```

test报错了，说明最新版本的`src.io/sampler` 是有问题的，我们查看一下它的所有可选版本。

```shell
# go list -m --versions rsc.io/sampler
rsc.io/sampler v1.0.0 v1.2.0 v1.2.1 v1.3.0 v1.3.1 v1.99.99
```

降级到v1.3.1试试

```shell
# go get rsc.io/sampler@v1.3.1
go: downloading rsc.io/sampler v1.3.1
# go test
PASS
ok  	github.com/hello	0.005s
```

所以说每一个通过`go get` 获取的包都能够加上版本标签，只不过一般默认情况下为`@latest`

#### 添加一个新的主版本依赖

添加一个新的函数，使用quote新版本中关于 `Go concurrency` 的功能

```go
# hello.go

package hello

import (
	"rsc.io/quote"
	quoteV3 "rsc.io/quote/v3"
) 


func Hello() string {
  return quote.Hello()
}

func Proverb() string {
	return quoteV3.Concurrency()
}
```

```go
# hello_test.go

package hello

import "testing"

func TestHello(t *testing.T) {
  want := "你好，世界。"
  if got := Hello(); got != want {
    t.Errorf("Hello() = %q, want %q", got, want)
  }
}

func TestProverb(t *testing.T) {
	want := "Concurrency is not parallelism."
  if got := Proverb(); got != want {
    t.Errorf("Hello() = %q, want %q", got, want)
  }
}
```

测试结果

```go
# go test

go: finding rsc.io/quote/v3 v3.1.0
go: downloading rsc.io/quote/v3 v3.1.0
go: extracting rsc.io/quote/v3 v3.1.0
PASS
ok      github.com/hello    0.024s
```

每Go module的一个大版本(v1,v2,...)都是用一个不同的路径，从v2版本开始，包路径最后必须添加版本号，比如rsc.io/quote的v3版本不再是r`sc.io/quote`,取而代之的是`rsc.io/quote/v3`. Go 允许同时使用多个主版本号不同的包，但是同一个大版本下面的小版本只能存在一个。

#### 将依赖直接更新到新的主版本

我们将程序中用到的 `rsc.io/quote` 直接升级到 `rsc.io/quote/v3`。因为每个大版本的升级都会伴随着各种API的调整，所以升级之前最好看看版本文档。

```go
# go doc rsc.io/quote/v3

package quote // import "rsc.io/quote/v3"

Package quote collects pithy sayings.

func Concurrency() string
func GlassV3() string
func GoV3() string
func HelloV3() string
func OptV3() string
```

可以看到`Hello`函数变成了`HelloV3`，更新代码看看结果

```go
# hello.go

package hello

import (
	"rsc.io/quote/v3"
) 


func Hello() string {
  return quote.HelloV3()
}

func Proverb() string {
	return quote.Concurrency()
}
```

```shell
# go test
PASS
ok  	github.com/hello	0.008s
```

### 删除不使用的依赖

我们移除了对 `src.io/quote` 的使用，但是 `go list -m all` 以及 `go.mod` 都还是显示存在。

```shel
# go list -m all
github.gom/gomodule
golang.org/x/text v0.3.2
golang.org/x/tools v0.0.0-20180917221912-90fa682c2a6e
rsc.io/quote v1.5.2
rsc.io/quote/v3 v3.1.0
rsc.io/sampler v1.3.1

# cat go.mod
module github.gom/hello

go 1.14

require (
	golang.org/x/text v0.3.2 // indirect
	rsc.io/quote v1.5.2
	rsc.io/quote/v3 v3.1.0
	rsc.io/sampler v1.3.1 // indirect
)
```

这是因为无论 `go build` 还是 `go test` 都能很容易的告诉你哪些包需要安装导入，但即使包已经不再使用，为了安全，它也不会主动去删除。

这时候我们需要使用 `go mod tidy` 来清除不再使用的包

```shell
# go mod tidy
# go list -m all
github.gom/gomodule
golang.org/x/text v0.3.2
golang.org/x/tools v0.0.0-20180917221912-90fa682c2a6e
rsc.io/quote/v3 v3.1.0
rsc.io/sampler v1.3.1

# cat go.mod
module github.com/hello

go 1.14

require (
	golang.org/x/text v0.3.2 // indirect
	rsc.io/quote/v3 v3.1.0
	rsc.io/sampler v1.3.1 // indirect
)

# go test
PASS
ok  	github.com/hello	0.008s
```

### 总结

Go modules 的大概工作流程为：

- `go mod init` 创建新的模块并生成go.mod文件
- `go build`, `go test` 以及其他的编译工具会将新的依赖添加到go.mod
- `go list -m all` 打印当前模块的所有依赖
- `go get` 根据版本号添加或者修改依赖
- `go mod tidy` 移除不再使用的依赖

### 相关阅读

- [Using Go Modules](https://blog.golang.org/using-go-modules)