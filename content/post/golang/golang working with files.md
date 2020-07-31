---
title: "Golang Working With Files"
date: 2020-07-27T14:38:57+08:00
tags: ["golang", "files"]
categories: ["golang"]
draft: false
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

### 判断读写权限

与上面的例子类似，检测权限的方法是`os.IsPermission()`

```go
package main

import (
	"log"
  "os"
)

func main(){
  // Test write permissions.It is possible the file 
  // does not exist and that will return a different
  // error that can be checked with os.IsNotExist(err)
  file, err := os.OpenFile("test.txt", os.O_WRONLY, 0666)
  if err != nil {
    if os.IsPermission(err) {
      log.Println("Error: Write permission denied.")
    }
  }
  file.Close()
  
  // Test read permissions
  file, err = os.OpenFile("test.txt", os.O_RDONLY, 0666)
  if err != nil {
    if os.IsPermission(err) {
      log.Println("Error: Read permission denied.")
    }
  }
  file.Close()
}
```

### 改变权限、所属、时间戳

如果你有正确的权限，你可以利用标准库提供的方法改变权限、所属、时间戳。用到的方法有：

- os.Chmod()
- os.Chown()
- os.Chtimes()

```go
package main 

import (
	"log"
  "os"
  "time"
)

func main(){
  // Change permissions using Linux style
  err := os.Chmod("test.txt", 0777)
  if err != nil {
    log.Println(err)
  } 
  
  // Change ownership
  err = os.Chown("test.txt", os.Getuid(), os.Getgid())
  if err != nil {
    log.Println(err)
  }
  
  // Change timestamps
  twoDaysFromNow := time.Now().Add(48 * time.Hour)
  lastAccessTime := twoDaysFromNow
  lastModifyTime := twoDaysFromNow
  err = os.Chtimes("test.txt", lastAccessTime, lastModifyTime)
  if err != nil {
    log.Println(err)
  }
}
```

### 硬链接和软链接

通常文件的本质是指向硬盘某块儿存储的指针，也称为inode。而硬链接就是创建一个新的指针指向同一块地方。只有当所有链接都被移除，一个文件才真正会被删除。硬链接只在同一个文件系统生效，我们可以将硬链接认为是'正常'链接。

符号链接或者也称为软链接与之略微不同，它不直接指向硬盘位置。符号链接只引用其他文件的名称，它可以指向不同文件系统的文件。不过需要注意的是，不是所有的系统都支持软链接。

Windows由于历史原因对软链接支持不够好，不过在Windows 10系统中，如果你获取了管理员权限，硬链接软链接都能很好的支持。

```go
package main

import (
	"fmt"
  "log"
  "os"
)

func main() {
  // Create a hard link
  // You will have two file names that point to the same contents
  // Changing the contents of one will change the other 
  // Deleting/renaming one will not affect the other
  err := os.Link("original.txt", "original_also.txt")
  if err != nil {
    log.Fatal(err)
  }
  
  fmt.Println("Creating symlink")
  // Create a symlink
  err = os.Symlink("original.txt", "original_sym.txt")
  if err != nil {
    log.Fatal(err)
  }
  
  // Lstat will return file info, but if it is actually 
  // a symlink, it will return info about the symlink.
  // It will not follow the link and give information
  // about the real file
  // Symlinks do not work in Windows
  fileInfo, err := os.Lstat("original_sym.txt")
  if err != nil{
    log.Fatal(err)
  }
  fmt.Printf("Link info: %+v", fileInfo)
  
  // Change ownership of a symlink only 
  // and not the file it points to
  err = os.Lchown("original_sym.txt", os.Getuid(), os.Getgid())
  if err != nil {
    log.Fatal(err)
  }
  
}
```



## 文件读写

文件的读写可以有多种方式，Go提供了相关的接口使得你能够方便的实现自己的方法来处理文件或者其他的读写接口。

你可以在`os`、`io`、`ioutil`包中找到需要的合适方法

### 拷贝文件

```go
package main

import (
	"io"
  "log"
  "os"
)

func main() {
  // Open original file 
  originalFile, err := os.Open("test.txt")
  if err != nil{
    log.Fatal(err)
  }
  defer originalFile.Close()
  
  // Create new file
  newFile, err := os.Create("test_copy.txt")
  if err != nil {
    log.Fatal(err)
  }
  defer newFile.Close()
  
  // Copy the bytes to destination from source
  bytesWritten, err := io.Copy(newFile, originalFile)
  if err != nil{
    log.Fatal(err)
  }
  log.Printf("Copied %d bytes.", bytesWritten)
  
  // Commit the file contents
  // Flushes memory to disk
  err = newFile.Sync()
  if err != nil{
    log.Fatal(err)
  }
  
}
```

### 通过位置在文件中查找

Seek() 函数可以用来将文件光标定位到特定位置。默认情况下的偏移位是0， 可以以bytes为单位向前读取。你也可以重置光标的位置到文件头部或者跳转到指定位置。

Seek()有两个参数，第一个参数表示移动距离，单位是bytes，正数表示向前移动，负数表示向后移动。第二个参数表示光标相对从那里开始，是相对偏移量的参考点。这个参数可以是0,1,2分别代表文件开头，当前位置，文件末尾。

举例来说，Seek(-1, 2)表示从文件末尾以1byte向头部移动；Seek(2,0)表示从文件头部向前2byte移动；Seek(5, 1)表示从当前位置向前每次移动5byte。

```go
package main

import (
	"fmt"
  "log"
  "os"
)

func main() {
  file, _ := os.Open("test.txt")
  defer file.Close()
  
  // Offset is how many bytes to move
  // Offset can be positive or negative
  var offset int64 = 5
  
  // Whence is the point of reference for offset
  // 0 = Beginning of file
  // 1 = Current of file
  // 2 = End of file
  var whence int = 0
  newPosition, err := file.Seek(offset, whence)
  if err != nil{
    log.Fatal(err)
  }
  fmt.Println("Just moved to 5: ", newPosition)
  
  // Go back 2 bytes from current position
  newPosition, err = file.Seek(-2, 1)
  if err != nil{
    log.Fatal(err)
  }
  fmt.Println("Just moved back two: ", newPosition)
  
  // Find the current position by getting the return value from Seek after moving 0 bytes
  currentPosition, err := file.Seek(0,1)
  fmt.Println("Current position:", currentPosition)
  
  // Go to beginning of file
  newPosition, err = file.Seek(0,0)
  if err != nil {
    log.Fatal(err)
  }
  fmt.Println("Position after seeking 0,0:", newPosition)
}
```

### 向文件写入bytes

向文件写入bytes可以使用最简单的`os`包，其他包`io`, `ioutil`, `bufio`都提供类似的方法。

```go
package main

import (
	"log"
  "os"
)

func main(){
  // Open a new file for writing only 
  file, err := os.OpenFile(
  	"test.txt",
    os.O_WRONLY|os.O_TRUNC|os.O_CREATE,
    0666,
  )
  if err != nil{
    log.Fatal(err)
  }
  defer file.Close()
  
  // Write bytes to file
  byteSlice := []byte("Bytes!\n")
  bytesWritten, err := file.Write(byteSlice)
  if err != nil {
    log.Fatal(err)
  }
  log.Printf("Wrote %d bytes.\n", bytesWritten)
}
```

### 快速写入文件

`ioutil`包提供了一个有用的函数`WriteFile()`，这个函数封装了底层操作，处理文件创建打开、写入bytes切片，并且关闭文件句柄。如果的需求是快速的将bytes写入文件，可以使用这个函数。

```go
package main

import (
	"io/ioutil"
  "log"
)

func main(){
  err := ioutil.WriteFile("test.txt", []byte("Hi\n"), 0666)
  if err != nil {
    log.Fatal(err)
  }
}
```

### 写入缓存

`bufio`包提供了缓存写入这样我们就能够在将数据写入硬盘之前先写入内存缓存。这样做的好处是，我们可以在写入硬盘之前对数据做很多操作，节省了磁盘IO时间。还有一个好处是能够缓存到一定量数据再一次性写入硬盘，还是减少磁盘IO的需求。举个极端的例子，假设我们每次写入1byte到文件，频繁的写硬盘会损伤硬盘寿命以及降低程序效率，我们可以先写入内存缓存(buffer)到达一定数量之后一次性写入文件。

写缓存可以检测当前有多少未缓存数据、多少缓存空间剩余。缓存可以重置当前操作到上次写入状态，缓存大小可扩展。

下面的例子演示了上面提到的理论

```go
package main

import (
	"bufio"
  "log"
  "os"
)

func main() {
  // Open file for writing 
  file, err := os.OpenFile("test.txt", os.O_WRONLY, 0666)
  if err != nil{
    log.Fatal(err)
  }
  defer file.Close()
  
  // Create a buffered writer from the file
  bufferedWriter := bufio.NewWriter(file)
  
  // Write bytes to buffer
  bytesWritten, err := bufferedWriter.Write(
    []byte{65, 66, 67},
  )
  if err != nil{
    log.Fatal(err)
  }
  log.Printf("Bytes written: %d\n", bytesWritten)
  
  // Write string to buffer
  // Also available are WriteRune() and WriteByte()
  bytesWritten, err = bufferedWriter.WriteString("Buffered string\n")
  if err != nil{
    log.Fatal(err)
  }
  log.Printf("Bytes written: %d\n", bytesWritten)
  
  // Check how much is stored in buffer waiting
  unflushedBufferSize := bufferedWriter.Buffered()
  log.Printf("Bytes buffered: %d\n", unflushedBufferSize)
  
  // See how much buffer is available
  bytesAvailable := bufferedWriter.Available()
  if err != nil{
    log.Fatal(err)
  }
  log.Printf("Available buffer: %d\n", bytesAvailable)
  
  // Write memory buffer to disk
  bufferedWriter.Flush()
  
  // Revert any changes done to buffer that have 
  // not yet been written to file with Flush()
  // We just flushed, so there are no changes to revert
  // The writer that you pass as an argument
  // is where the buffer will output to, if you want
  // to change to a new writer
  bufferedWriter.Reset(bufferedWriter)
  
  // See how much buffer is available
  bytesAvailable = bufferedWriter.Available()
  if err != nil{
    log.Fatal(err)
  }
  log.Printf("Available buffer: %d\n", bytesAvailable)
  
  // Resize buffer. The first argument is a writer
  // where the buffer should output to.In this case
  // we are using the same buffer. If we chose a number
  // that was smaller than the existing buffer, like 10
  // we would not get back a buffer of size 10, we will
  // get back a buffer the size of the original since
  // it was already large enough (default 4096)
  bufferedWriter = bufio.NewWriterSize(bufferedWriter, 8000)
  
  // Check available buffer size after resizing
  bytesAvailable = bufferedWriter.Available()
  if err != nil{
    log.Fatal(err)
  }
  log.Printf("Available buffer: %d\n", bytesAvailable)
}

```

### 从文件读取bytes

`os.File`类型有很多基础方法，比如`File.Read()`它接收一个bytes类型的切片作为参数，从文件读取bytes数据存储在切片中。`Read()`会读取尽可能多的数据知道接收容器被填满。

可以多次调用`Read()`函数来读取到文件末尾，如果到达末尾`io.EOF`错误会被返回。

```go
package main

import (
	"log"
  "os"
)

func main() {
  // Open file for reading 
  file, err := os.Open("test.txt")
  if err != nil {
    log.Fatal(err)
  }
  defer file.Close()
  
  // Read up to len(b) bytes from the File
  // Zero bytes written means end of file
  // End of file returns error type io.EOF
  bytesSlice := make([]byte, 16)
  bytesRead, err := file.Read(bytesSlice)
  if err != nil{
    log.Fatal(err)
  }
  log.Printf("Number of bytes read: %d\n", bytesRead)
  log.Printf("Data read: %s\n", bytesSlice)
}
```

#### 读取准确的bytes

前面介绍的`File.Read()`函数不会报错，当你提供的切片大小为500bytes，而文件仅有10bytes。如果你的需求是想要整个指定大小buffer全部被填满。可以使用`io.ReadFull()`，这个函数会在填充数据不够的时候报错，如果没有数据读取，它会返回EOF错误；如果读了一部分数据就遭遇EOF，它会返回`ErrUnexpectedEOF`错误

```go
package main

import (
	"io"
  "log"
  "os"
)

func main() {
  // Open file for reading 
  file, err := os.Open("test.txt")
  if err != nil{
    log.Fatal(err)
  }
  
  // The file.Read() function will happily read a tiny file in to a
  // large byte slice, but io.ReadFull() will return an
  // error if the file is smaller than the byte slice
  byteSlice := make([]byte, 2)
  numBytesRead, err := io.ReadFull(file, byteSlice)
  if err != nil{
    log.Fatal(err)
  }
  log.Printf("Number of bytes read: %d\n", numBytesRead)
  log.Printf("Data read: %s\n", byteSlice)
}
```

#### 最少读取 N bytes

`io.ReadAtLeast()`会报错，如果不能够提供最小数量的bytes。

```go
package main

import (
	"io"
  "log"
  "os"
)

func main() {
  // Open file for reading 
  file, err := os.Open("test.txt")
  if err != nil{
    log.Fatal(err)
  }
  
  byteSlice := make([]byte, 512)
  minBytes := 8
  // io.ReadAtLeast() will return an error if it cannot 
  // find at least minBytes to read. Ti will read as 
  // many bytes as byteSlice can hold.
  numBytesRead, err := io.ReadAtLeast(file, byteSlice, minBytes)
  if err != nil {
    log.Fatal(err)
  }
  log.Printf("Number of bytes read: %d\n", numBytesRead)
  log.Printf("Data read: %s\n", byteSlice)
}
```

#### 读取文件的所有bytes

`ioutil`包提供了一个方法，能够一次性读取文件所有byte并返回byte切片，这样做的便利之处就是你不用在读之前提前定义。不便之处就是如果文件特别大，它会一次性返回大到无法想象的切片。

```go
package main

import (
	"fmt"
  "io/ioutil"
  "log"
  "os"
)

func main(){
  // Open file for reading 
  file, err := os.Open("test.txt")
  if err != nil {
    log.Fatal(err)
  }
  
  // os.File.Read(), io.ReadFull() and 
  // io.ReadAtLeast() all work with a fixed
  // byte slice that you make before you read
  
  // ioutil.ReadAll() will read every byte 
  // from the reader (in this case a file)
  // and return a slice of unknown slice  
  data, err := ioutil.ReadAll(file)
  if err != nil{
    log.Fatal(err)
  }
  
  fmt.Printf("Data as hex: %x\n", data)
  fmt.Printf("Data as string: %s\n", data)
  fmt.Println("Number of bytes read:", len(data))
}
```

#### 读取文件到内存

更前面的`io.ReadAll()`类似，`io.ReadFile()`也是将文件读取返回byte切片，不同的是，后者参数是文件路径，而不是打开的文件对象。这个函数会管理文件的打开、读取、关闭。这几乎是最方便的读取文件的方法。

没有十全十美，这个方法的缺点就是会将全部文件直接读取到内存，大文件会比较占内存。

```go
package main

import (
  "io/ioutil"
  "log"
)

func main(){
  // Read file to byte silce
  data, err := ioutil.ReadFile("test.txt")
  if err != nil{
    log.Fatal(err)
  }
  log.Printf("Data read: %s\n", data)
}
```

### 缓存读取器(Buffered reader)

创建一个缓存读取器会在内存缓存区存储一些内容，它提供一些在`os.File``io.Reader`这些类型不具备的方法。默认buffer大小是4096最小大小是16。

缓存读取器提供的方法有:

- Read(): 读取数据到byte slice
- Peek():在不移动文件光标的前提下读取下一个bytes
- ReadByte(): 读取一个byte
- UnreadByte(): 取消读取最后一个byte
- ReadBytes(): 读取bytes直到遇到指定分隔符
- ReadString(): 读取字符串直到遇到指定分隔符

下面的例子演示了怎么使用buffered reader 从文件中获取数据。

```go
package main

import (
	"bufio"
  "fmt"
  "log"
  "os"
)

func main(){
  // Open file and create a buffered reader on top
  file, err := os.Open("test.txt")
  if err != nil{
    log.Fatal(err)
  }
  bufferedReader := bufio.NewReader(file)
  
  // Get bytes without advancing pointer
  byteSlice := make([]byte, 5)
  byteSlice, err = bufferedReader.Peek(5)
  if err != nil{
    log.Fatal(err)
  }
  fmt.Printf("Peeked at 5 bytes: %s\n", byteSlice)
  
  // Read and advance pointer
  numBytesRead, err := bufferedReader.Read(byteSlice)
  if err != nil{
    log.Fatal(err)
  }
  fmt.Printf("Read %d bytes: %s\n", numBytesRead, byteSlice)
  
  // Ready 1 byte.Error if no byte to read
  myByte, err := bufferedReader.ReadByte()
  if err != nil{
    log.Fatal(err)
  }
  fmt.Printf("Read 1 byte: %c\n", myByte)
  
  // Read up to and including delimiter
  // Returns byte slice
  dataBytes, err := bufferedReader.ReadBytes('\n')
  if err != nil{
    log.Fatal(err)
  }
  fmt.Printf("Read bytes: %s\n", dataBytes)
  
  // Read up to and including delimiter
  dataString, err := bufferedReader.ReadString('\n')
  if err != nil{
    log.Fatal(err)
  }
  fmt.Printf("Read string: %s\n", dataString)
  // This example reads a few lines so test.txt
  // should have a few lines of text to work correct
  
}
```

### 使用 scanner 读取文件

Scanner 属于`bufio`包， 它对于按照特定的分隔符逐步遍历文件非常有用。通常来说我们通过换行符来将文件分割为多行。在CSV中，逗号是分隔符。`bufio.Scanner`可以包装`os.File`对象 ，使用`Scan()`来读取到下一个分隔符的内容。然后使用`Text()`或者'Bytes()'来获取得到的数据。

这里的分隔符不是简单的byte或者单个字符，需要你自己实现一个特殊的方法，决定分隔符在哪里，每次移动多少指针，以及返回什么数据。如果没有自定义`SplitFunc`，系统提供了默认的`ScanLines`，以及`ScanRunes` `ScanWords`

自定义分割函数，需要实现类似的函数签名

```go
type SplitFuncfunc(data []byte, atEOF bool) (advance int, token []byte, err error)
```

```go
package main

import (
	"bufio"
  "fmt"
  "log"
  "os"
)

func main(){
  // Open file and create scanner on top of it
  file, err := os.Open("test.txt")
  if err != nil{
    log.Fatal(err)
  }
  scanner := bufio.NewScanner(file)
  
  // Default scanner is bufio.ScanLines.Lets use ScanWords.
  // Could also use a custom function of SplitFunc type
  scanner.Split(bufio.ScanWords)
  // Scan for next token.
  success := scanner.Scan()
  if success == false {
    // False on error or EOF.Check error
    err = scanner.Err()
    if err == nil{
      log.Println("Scan completed and reached EOF")
    }else{
      log.Fatal(err)
    }
  }
  // Get data from scan with Bytes() or Text()
  fmt.Println("First word found: ", scanner.Text())
  
  // Call scanner.Scan() manually, or loop with for
  for scanner.Scan() {
    fmt.Println(scanner.Text())
  }
}
```



## 文件归档

归档是一种文件格式用来存储多数文件。最常用的两种归档格式是`tar balls` `zip archives` Go标准库包含`tar`和`zip`包。

### 利用ZIP打包文件

```go
// This example uses zip but standard library
// also supports tar archives
package main

import (
	"archive/zip"
  "log"
  "os"
)

func main(){
  // Create a file to write the archive buffer to 
  // Could also use an in memory buffer.
  outFile, err := os.Create("test.zip")
  if err != nil{
    log.Fatal(err)
  }
  defer outFile.Close()
  
  // Create a zip writer on top of the file writer
  zipWriter := zip.NewWriter(outFile)
  
  // Add files to archive
  // We use some hard code data to demonstrate,
  // but you could iterate through all the files
  // of each file, or you can take data from your
  // program  and write it in to the archive
  var filesToArchive = []struct{
    Name, Body string
  }{
    {"test.txt", "String contents of file"},
    {"test2.txt", "\x61\x62\x63\n"},
  }
  
  // Create and write files to the archive, which in turn 
  // are getting written to the underlying writer to the
  // .zip file we created at the beginning
  for _, file := range filesToArchive {
    fileWriter, err := zipWriter.Create(file.Name)
    if err != nil{
      log.Fatal(err)
    }
    _, err = fileWriter.Write([]byte(file.Body))
    if err != nil {
      log.Fatal(err)
    }
  }
  
  // Clean up
  err = zipWriter.Close()
  if err != nil {
    log.Fatal(err)
  }
  
}
```

### 利用unzip解压

```go
// This example uses zip but standard library
// also supports tar archlives
package main

import (
  "archive/zip"
  "io"
  "log"
  "os"
  "path/filepath"
)

func main() {
  // Create a reader out of the zip archive
  zipReader, err := zip.OpenReader("test.zip")
  if err != nil {
    log.Fatal(err)
  }
  defer zipReader.Close()
  
  // Iterate through each file/dir found in
  for _, file := range zipReader.Reader.File{
    // Open the file inside the zip archive
    // like a normal file
    zippedFile, err := file.Open()
    if err != nil{
      log.Fatal(err)
    }
    defer zippedFile.Close()
    
    // Specify what the extracted file name should be.
    // You can specify a full path or a prefix
    // to move it to a different directory.
    // In this case, we will extract the file from
    // the zip to a file of the same name.
    targetDir := "./"
    extractedFilePath := filepath.Join(
    	targetDir,
      file.Name,
    )
    
    // Extract the item (or create directory)
    if file.FileInfo().IsDir(){
      // Create directories to recreate directory
      // structure inside the zip archive.Also 
      // preserves permissions
      log.Println("Creating directory:", extractedFilePath)
      os.MkdirAll(extractedFilePath, file.Mode())
    } else {
      // Extract regular file since not a directory
      log.Println("Extracting file:", file.Name)
      
      // Open an output file for writing
      outputFile, err := os.OpenFile(
      	extractedFilePath,
        os.O_WRONLY|os.O_CREATE|os.O_TRUNC,
        file.Mode(),
      )
      if err != nil{
        log.Fatal(err)
      }
      defer outputFile.Close()
      
      // "Extract" the file by copying zipped file
      // contents to the output file
      _, err := io.Copy(outputFile, zippedFile)
      if err != nil{
        log.Fatal(err)
      }
    }
  }
}
```



## 文件压缩

Go标准库支持压缩，包括的算法有：

- bzip2: bzip2 format
- flate: DEFLATE(RFC 1951)
- gzip: gzip format(RFC 1952)
- lzw: Lempel-Ziv-Welch format
- zlib: zlib format(RFC 1950)

相关的官方文档可以访问 https://golang.org/pkg/compress/

### 压缩文件

```go
// This example uses gzip but standard library alse
// supports zlib, bz2, flate, and lzw
package main

import (
	"compress/gzip"
  "log"
  "os"
)

func main(){
  // Create .gz file to write to 
  outputFile, err := os.Create("test.txt.gz")
  if err != nil{
    log.Fatal(err)
  }
  
  // Create a gzip writer on top of file writer
  gzipWriter := gzip.NewWriter(outputFile)
  defer gzipWriter.Close()
  
  // When we write to the gzip writer
  // it will in turn compress the contents
  // and then write it to the underlying
  // file writer as well
  // We don't have to worry about how all
  // the compression works since we just
  // use it as a simple writer interface
  // that we send bytes to
  _, err := gzipWriter.Write([]byte("Gophers rule!\n"))
  if err != nil {
    log.Fatal(err)
  }
  log.Println("Compressed data written to file.")
}
```

### 解压缩文件

```go
// This example uses gzip but standard library alse
// supports zlib, bz2, flate, and lzw
package main

import (
	"compress/gzip"
  "io"
  "log"
  "os"
)

func main(){
  // Open gzip file that we want to uncompress
  // The file is a reader, but we could use any
  // data source.It is common for web servers
  // to return gzipped contents to save bandwidth
  // and in that case the data is not in a file
  // on the file system but is in a memory buffer
  gzipFile, err := os.Open("test.txt.gz")
  if err != nil {
    log.Fatal(err)
  }
  
  // Create a gzip reader on top of the file reader
  // Again, it could be any type reader though
  gzipReader, err := gzip.NewReader(gzipFile)
  if err != nil {
    log.Fatal(err)
  }
  defer gzipReader.Close()
  
  // Uncompress to a writer. We'll use a file writer
  outfileWriter, err := os.Create("unzipped.txt")
  if err != nil {
    log.Fatal(err)
  }
  defer outfileWriter.Close()
  
  // Copy contents of gzipped file to ouput file
  _, err = io.Copy(outfileWriter, gzipReader)
  if err != nil {
    log.Fatal(err)
  }
}
```



## 创建临时文件及文件夹

`ioutil`包提供了两个方法`TempDir()`和`TempFile()` 这两个方法默认路径使用系统的临时文件夹(比如LInux 的 /tmp)

```go
package main

import (
  "fmt"
  "io/ioutil"
  "log"
  "os"
)

func main() {
  // Create a temp dir in the system default temp folder
  tempDirPath, err := ioutil.TempDir("", "myTempDir")
  if err != nil{
    log.Fatal(err)
  }
  fmt.Println("Temp dir created:", tempDirPath)
  
  // Create a file in new temp directory
  tempFile, err := ioutil.TempFile(tempDirPath, "myTempFile.txt")
  if err != nil {
    log.Fatal(err)
  }
  fmt.Println("Temp file created:", tempFile.Name())
  
  // ...do something with temp file/dir ....
  
  // Close file
  err = tempFile.Close()
  if err != nil{
    log.Fatal(err)
  }
  
  // Delete the resources we created
  err = os.Remove(tempFile.Name())
  if err != nil{
    log.Fatal(err)
  }
  err = os.Remove(tempDirPath)
  if err != nil{
    log.Fatal(err)
  }
  
}
```



## 通过HTTP下载文件

下面的例子是下载特定URL资源到文件

```go
package main

import (
  "io"
  "log"
  "net/http"
  "os"
)

func main(){
  // Create output file
  newFile, err := os.Create("devdungeon.html")
  if err != nil{
    log.Fatal(err)
  }
  defer newFile.Close()
  
  // HTTP GET request devdungeon.com
  url := "http://www.devdungeon.com/archive"
  response, err := http.Get(url)
  defer response.Body.Close()
  
  // Write bytes from HTTP response to file
  // response.Body satisfies the reader interface.
  // newFile satisfies the writer interface.
  // That allows us to use io.Copy which accepts
  // any type that implements reader and writer interface
  numBytesWritten, err := io.Copy(newFile, response.Body)
  if err != nil{
    log.Fatal(err)
  }
  
  log.Printf("Downloading %d byte file.\n", numBytesWritten)
}
```



## 总结

这篇文章总结了操作文件的一些常用方法，让我们了解到都有哪些方法可以使用。这里面的代码可以作为例子帮助你记起相关操作。

这些方法分布到了不同的包中。其中`os`包只提供了最基础的操作包括文件的打开,关闭简单的读取。而`io`包提供的方法能够用于实现了reader、writer接口的数据，这要比`os`包更高阶一些。最后的`ioutil`包提供了更加高阶并且封装好的便捷方法供我们使用。



