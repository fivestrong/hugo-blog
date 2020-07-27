---
title: "Golang Working With Files"
date: 2020-07-27T14:38:57+08:00
tags: []
categories: ["golang"]
draft: true
---

学习目标:

- 创建空文件以及截取文件
- 获取文件详细信息
- 重命名、移动、删除文件
- 操作文件权限、所属、以及时间戳
- 文件符号链接
- 多种读写文件的方式
- 文件归档
- 文件压缩
- 临时文件以及文件夹创建
- 通过HTTP下载文件

## 文件基础

### 创建空文件

我们假设有这样一个场景：有一台服务器每天滚动写入日志，日志名称由当天时间决定。一般情况下开发者会将权限设置周全，比如说只有管理员才可以读取日志数据。但是有一种可能他们对目录的权限限制比较弱，这时候假设我们提前创建下一天的文件，一般情况下服务仅仅当文件不存在的时候才会创建特定权限的新文件，但是如果检测文件已经存在就不会检测权限而直接写入。这样我们就可以利用这个漏洞创建有自己可读权限的日志文件。这个文件的命名格式和服务的要一致，比如，如果服务使用的格式是`logs-2020-07-21.txt`，那我们就应该创建`logs-2020-07-22.txt`.这样明天服务就会写入这个你有权限的文件，而不是生成仅仅root才能读写的日志。

代码示例：

```go
package main

import (
	"log"
  "os"
)

func main(){
  newFile, err := os.Create("test.txt")
  if err != nil {
    log.Fatal(err)
  }
  log.Println(newFile)
  newFile.Close()
}
```

### 截断文件

截断文件是指将文件修剪到指定大小。常用于完全将文件内容移除，也可以用于将文件限制到特定大小。`os.Truncate()`函数的一个显著特点是当目标文件小于需要截断的大小时，它会将文件空白部分使用空bytes填充到指定长度。

截断文件比创建空文件还要广泛一些。当日志文件太大，我们可以通过截取文件将它保存到硬盘上。如果你是个黑客，你可能会想要通过截取方式掩盖`bash_history`以及其他日志文件的记录。

```go
package main

import (
	"log"
  "os"
)

func main(){
  // Truncate a file to 100 bytes.If file
  // is less than 100 bytes, and the rest of the space is 
  // filled with null bytes. If it is over 100 bytes,
  // Everything past 100 bytes will be lost.Either way
  // we will end up with exactly 100 bytes.
  // Pass in 0 to truncate to a completely empty file
  
  err := os.Truncate("test.txt", 100)
  if err != nil {
    log.Fatal(err)
  }
}
```

### 获取文件信息

下面的例子会获取文件能够得到的元数据。其中包括属性，名称、权限、最后一次修改时间、是否是文件夹等等信息。

```go
package main

import (
	"fmt"
  "log"
  "os"
)

func main(){
  // Stat returns file info.It will return an error if there is no file.
  fileInfo, err := os.Stat("test.txt")
  if err != nil {
    log.Fatal(err)
  }
  fmt.Println("File name: ", fileInfo.Name())
  fmt.Println("Size in bytes: ", fileInfo.Size())
  fmt.Println("Permissions: ", fileInfo.Mode())
  fmt.Println("Last modified:", fileInfo.ModTime())
  fmt.Println("Is Directory: ", fileInfo.IsDir())
  fmt.Printf("System interface type: %T\n ", fileInfo.Sys())
  fmt.Printf("System info: %+v\n\n ", fileInfo.Sys())
}
```

### 重命名文件

本质上重命名和移动文件性质是一致的，所以如果你打算将一个文件移动到另外一个文件夹，可以使用 `os.Rename()`函数

```go
package main

import (
	"log"
  "os"
)

func main (){
  originalPath := "test.txt"
  newPath := "test2.txt"
  err := os.Rename(originalPath, newPaht)
  if err != nil {
    log.Fatal(err)
  }
}
```

### 删除文件

```go
package main 

import (
	"log"
  "os"
)

func main(){
  err := os.Remove("test.txt")
  if err != nil {
    log.Fatal(err)
  }
}
```

### 打开关闭文件

打开文件有很多种方式：

os.Open()函数传入文件名，得到一个只读文件。os.OpenFile()函数可以传入更多参数，你可以指定文件的可读可写权限，也可以指定打开文件方式是读，写，追加，不存在创建或是根据需求截取。

关闭文件只需要调用`Close()`方法，可以立马调用关闭，也可以使用`defer`延迟关闭。

```go
package main

import (
	"log"
  "os"
)

func main(){
  // Simple read only open.We will cover actually reading 
  // and writing to files in examples further down the page
  file, err := os.Open("test.txt")
  if err != nil {
    log.Fatal(err)
  }
  file.Close()
  
  // OpenFile with more optinos. Last param is the permission mode
  // Second param is the attributes when opening
  file, err = os.OpenFile("test.txt", os.O_APPEND, 0666)
  if err != nil {
    log.Fatal(err)
  }
  file.Close()
  
  // Use these attributes individually or combined
  // with an OR for second arg os OpenFile()
  // e.g. os.O_CREATE|os.O_APPEND
  // OR os.O_CREATE|os.O_TRUNC|os.O_WRONLY
  
  // os.O_RDONLY // Read only
  // os.O_WRONLY // Write only
  // os.O_RDWR   // Read and write
  // os.O_APPEND // Append to end of file
  // os.O_CREATE // Create is none exist
  // os.O_TRUNC  // Truncate file when opening
}
```

### 判断文件是否存在

判断文件是否存在分为两个步骤：

首先，通过`os.Stat()`函数获取到文件的`FileInfo`对象。如果文件不存在，返回结果就是一个错误类型。所以文件信息获取一定要记得检查返回错误。

标准库中提供了`os.IsNotExist()`它能够通过检测返回错误判断文件是否存在。

```go
package main

import (
	"log"
  "os"
)

func main() {
  // Stat returns file info. It will return 
  // an error if there is no file
  fileInfo, err := os.Stat("test.txt")
  if err != nil {
    if os.IsNotExist(err) {
      log.Fatal("Flie does not exist.")
    }
  }
  log.Println("File does exist. File information: ")
  log.Println(fileInfo)
}
```



## 文件读写



## 文件归档



## 文件压缩



## 创建临时文件及文件夹



## 通过HTTP下载文件



## 总结





