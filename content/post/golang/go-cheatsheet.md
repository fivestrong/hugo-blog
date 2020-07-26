---
title: "Go Cheatsheet"
date: 2020-07-25T15:31:16+08:00
tags: ["cheatsheet"]
categories: ["golang"]
draft: false
---

学习目标：了解Go的基本内容，方便复习查看。

## 关键词(Keywords)

Go 总共只有25歌关键词，大多数与其他语言类似。

### Data types

|        |                               |
| :-------- | ---------------------------------------------- |
| var       | 用于定义新的变量                               |
| const     | 用于定义常量                                   |
| type      | 用于定义新的数据类型                           |
| struct    | 用于定义新的结构类型，这种类型能够包含多个变量 |
| map       | 用于定义map或者hash变量                        |
| Interface | 用于定义新的接口                               |

### Functionos

|        | 		                            |
| ------ | ------------------------------ |
| func   | 定义新的函数                   |
| return | 退出函数（可选功能，返回变量） |

### Packages

|   |    |
| ------- | ---------------------- |
| import  | 向当前包中导入其他包   |
| package | 定义当前文件属于哪个包 |

### Program flow

|      |                                      |
| ---- | -------------------------------------- |
| if   | 当条件为true时候，执行分支语句         |
| else | 当条件为false时候，执行这个分支语句    |
| goto | 用于直接跳转到标签，很少使用并且不推荐 |

### Switch statements

|             |                               |
| ----------- | -------------------------------- |
| switch      | 用于根据条件进行分支             |
| case        | 为switch申明定义条件             |
| default     | 当没有case匹配的时候默认执行语句 |
| fallthrough | 穿透执行下一个case               |

### Iteration

|          |                       |
| -------- | ------------------------------------------------------------ |
| for      | 这里的for关键词类似于c语言的for,可以提供：初始化语句，条件判断，自增语句。Go没有while循环，for关键字同时拥有for,while的功能。 |
| range    | 这个关键字和for一起用于迭代map或者slice>                     |
| continue | 跳过当前循环剩下所有的执行语句并直接执行下一次循环           |
| break    | 完全结束循环，即便还有循环没有执行完毕也跳过。               |

### Concurrency

|        |  |
| ------ | ------------------------------------------------------------ |
| go     | 语言内置轻量级线程,当调用函数的时候使用Go关键字，Go会在一个单独的线程中执行这个函数 |
| chan   | channels用于多个线程通信,用于发送接收特定类型数据，channels默认是阻塞的 |
| select | select 关键词能够通过非阻塞的方式来使用channels              |

### Convenience

|   |   |
| ----- | ------------------------------------------------------------ |
| defer | 这个关键词比较特殊，它能够让方法延迟执行。常用于文件或者数据流的延迟关闭。 |

## 代码

Go源码应该使用`.go`作为扩展名，默认支持UTF-8编码，这就意味着你可以在源文件中使用任何的Unicode字符，甚至是emoji表情包。

不像java，go语言每行代码结尾分号是可选的。但是如果你打算将多个声明或者表达式写到一行，那么分号是必须要求的。

Go 提供了标准的代码格式化功能，使用go fmt 可以格式化源代码

## 注释(comments)

Go注释类似于C++风格

```go
// 单行注释，在后面的语句全部被忽略
/* 这个注释可以注释多行代码 */
```

## 类型(Types)

### Booleen

Booleen类型代表true or false 值

```go
var customFlag bool = false //布尔值定义
```

### Numeric

#### generic numbers

通用数字适用于当你不关心它是否是32 or 64bits，它会根据处理器的位数来自动确定类型的最大值。

- uint: 32或者64bits 无符号整型
- int: 有符号整型大小与uint一致
- uintptr: 无符号整型用来存储指针值

#### specific numbers

unsigned integers

- nt8: 8-bit 
- uint16: 16-bit
- uint32: 32-bit
- uint64: 64-bit

signed integers

- int8: 8-bit
- int16: 16-bit
- int32: 32-bit
- int64: 64-bit

floating point numbers

- float32: 32-bit
- float64: 64-bit

other numeric types

- complex64: 以float32作为实部虚部
- complex128: 以float64作为实部虚部
- byte: alias for uint8
- rune: alias for int32

整数可以有多种方式表示，一般会通过十进制，八进制，十六进制等方式来定义。

```go
package main
import "fmt"
func main(){
  // Decimal for 15
  number0 := 15
  
  // Octal for 15
  number1 := 017
  
  // Hexadecimal for 15
  number2 := 0x0F
  fmt.Println(number0, number1, number2)
}
```

### String

Go 拥有一个`strings` 的包，里面有很多针对**string**类型的方法。还有一个`strconv`包，专门用于字符串到其他类型的转换。

字符串使用双引号定义，单引号一般用于单个字符或者runes类型。你可以使用`(backticks)符号包裹，来定义多行字符串。

```go
package main
import "fmt"
fun main(){
  // Long form assigment
  var myText = "test string 1"
  
  // Short form assignment
  myText2 := "test string 2"
  
  // Multiline string
  myText3 := `long string
  spanning multiple
  lines`
  
  fmt.Println(myText)
  fmt.Println(myText2)
  fmt.Println(myText3)
}
```

### Array

Arrays 是由特定类型（任意类型）的元素序列组成，数组的长度无法改变并且由初始化时候指定。

创建 128bytes数组的语法

```go
var myByteArray [128]byte
```

数组元素可以根据下标获取

```go
singleByte := myByteArray[4]
```

 

### Slice

切片底层使用的数据结构类型为array，切片的一大优点就是可扩展性，它的capacity属性代表底层array的大小，length属性代表当前切片可以使用的长度。

slice使用make函数创建，可以指定length和capacity的大小。

```go
make([]T, lengthAndCapacity) // 一个参数会定义length和capacity相同
make([]T, length, capacity) // 两个参数可以定义length和capacity不同，其中capacity可以比length大
```

如果一个切片的capacity和length为0，这个切片指向nil，并且底层不会有array数组。

切片定义例子：

```go
package main
import "fmt"
func main(){
  // Create  a nil slice
  var mySlice []byte
  
  // Create a byte slice of length 8 and max capacity 128
  mySlice = make([]byte, 8, 128)
  
  // Maximum capacity of the slice
  fmt.Println("Capacity: ", cap(mySlice))
  
  // Current length of slice
  fmt.Println("Length: ", len(mySlice))
}
```

可以使用内置的`append()`函数向slice追加内容，可以同时追加一个或多个数据。如果超过最大的capacity，底层array会自动扩容。但是需要了解的实这种扩容是新建一个array，将现有内容复制到新的array，如果当前slice内容很多，这个操作是很耗时的。所以语言本身会扩容到一个合适长度，以供增长需求。

切片使用例子：

```go
package main
import "fmt"
func main(){
  var mySlice []int // nil slice
  
  // Appending works on nil slices.
  // Since nil slices have zero capacity, and have no underlying array, it will create one.
  mySlice = append(mySlice, 1,2,3,4,5)
  
  // Individual elements can be accessed from a slice just like an array by using the square bracker operator.
  firstElement := mySlice[0]
  fmt.Println("First element: ", firstElement)
  
  // To get only the second and third element, use:
  subset := mySlice[1:4]
  fmt.Println(subset)
  
  // To get the full contents of a slice except for the first element, use:
  subset = mySlice[1:]
  fmt.Println(subset)
  
  // To get the full contents of a slice except for the last element, use:
  subset = mySlice[0:len(mySlice)-1]
  fmt.Println(subset)
  
  // To copy a slice, use the copy() function.
  // if you assign one slice to another with the equal operator,
  // the slices will point at the same memory location
  // and changing one would change both slices.
  slice1 := []int{1,2,3,4}
  slice2 := make([]int, 4)
  
  // Create a unique copy in memory
  copy(slice2, slice1)
  
  // Changing one should not affect the other
  slice2[3] = 99
  fmt.Println(slice1)
  fmt.Println(slice2)
}
```

### Struct

Go语言中struct或者数据结构可以包含众多不同类型的变量。

Go 使用大小写来定义变量或者方法的可见性，首字母大写类似其他语言的public，可以在其他包中被导入；首字母小写，类似private，只能在同一个包中被使用。

Go struct 使用例子

```go
package main 
import "fmt"
func main(){
  // Define a Person type.Both fields public
  type Person struct{
    Name string
    Age int
  }
  // Create a Person object and store the pointer to it.
  nanodano := &Person{Name: "NanoDano", Age: 99}
  fmt.Println(nanodano)
  
  // Structs can also be embedded within other structs.
  // This replaces inheritance by simply storing the data type as another variable.
  type Hacker struct {
    Person Person
    FavoriteLanguage string
  }
  hacker := &Hacker{
    Person: *nanodano,
    FavoriteLanguage: "Go",
  }
  fmt.Println(hacker)
  fmt.Println(hacker.Person.Name)
}
```

### Pointer

Go 提供指针类型来存储特定类型数据的内存位置。指针可以用来在函数中传参，这样传递的是一个引用而不是数据的拷贝。这样函数就能够在原地直接修改对象。

Go语言中的指针是安全的，它不允许对指针进行算数操作，所以这个指针只能作用于引用的已存在对象上。

指针使用例子：

```go
package main
import (
  "fmt"
  "reflect"
)
func main(){
  myInt := 42
  intPointer := &myInt
  fmt.Println(reflect.TypeOf(intPointer))
  fmt.Println(intPointer)
  fmt.Println(*intPointer)
}
```

### Function

Go 函数使用`func`关键字定义，可以定义多个参数，所有参数均为位置参数，Go没有命名参数。Go同时也支持可变参数。

Go支持多值返回， 可以使用`_`来忽略返回值。

函数例子：

```go
package main
import "fmt"
// Function with no parameters
func sayHello(){
  fmt.Println("Hello.")
}

// Function with one parameters
func greet(name string){
  fmt.Printf("Hello, %s.\n", name)
}

// Function with multiple params of same type
func greetCustom(name, greeting string){
  fmt.Printf("%s, %s.\n", greeting, name)
}

// Variadic parameters, unlimited parameters
func addAll(numbers ...int) int {
  sum := 0
  for _, number := range numbers {
    sum += number
  }
  return sum
}

// Function with multiple return values
// Multiple values encapsulated by parenthesis
func checkStatus()(int, error){
  return 200, nil
}

// Define a type as a function so it can be used
// as a return type
type greeterFunc func(string)
// Generate and return a function
func generateGreetFunc(greeting string) greeterFunc {
  return func(name string){
    fmt.Printf("%s, %s.\n", greeting, name)
  }
}

func main(){
  sayHello()
  greet("NanoDano")
  greetCustom("NanoDano", "Hi")
  fmt.Println(addAll(4,5,2,3,9))
  
  chineseGreet := generateGreetFunc("你好")
  chineseGreet("NanoDano")
  
  statusCode, err := checkStatus()
  fmt.Println(statusCode, err)
}
```

### Interface

接口是一种特殊的类型，它定义了一系列函数签名。你可以把接口理解为，为了满足这个接口类型，你的类型必须实现方法X和Y。这样在任何使用这个接口的地方，你可以使用自己的类型代替。这种声明方式是隐式的，你不用在定义类型的时候显示声明满足特定接口，编译器会自动检测。

为了满足接口，你必须实现它规定的函数，但是除此之外，你可以添加任意多的自定义函数。

使用比较多的一个接口是`error`，它仅仅需要一个实现函数，定义如下

```go
type error interface {
  Error() string
}
```

接下来我们通过这个接口来自定义一个：

```go
package main
import "fmt"
// Define a custom type that will be used to satisfy the error interface
type customError struct {
  Message string
}

// Satisfy the error interface
// by implementing the Error() function
// which return a string
func (e *customError) Error() string {
  return e.Message 
}

// Sample function to demonstrate
// how to use thee custom error
func testFunction() error {
  if true != false { // Mimic an error condition
    return &customError{"Something went wrong."}
  }
  return nil
}

func main(){
  err := testFunction()
  if err != nil {
    fmt.Println(err)
  }
}
```

### Map

map 是一个hash table 或者 存储键值对的字典。键值对可以是任何数据类型，包括map自己。

map是无序列的，每次调用显示的结果均不一样。并且maps不是进程安全的，如果非要在多个进程中使用map，可以使用mutex。

map使用例子：

```go
package main
import (
  "fmt"
  "reflect"
)

func main(){
  // Nil maps will cause runtime panic if used
  // without being initialized with make()
  var intToStringMap map[int]string
  var stringToIntMap map[string]int
  fmt.Println(reflect.TypeOf(intToStringMap))
  fmt.Println(reflect.TypeOf(stringToIntMap))
  
  // Initialize a map using make
  map1 := make(map[string]string)
  map1["Key Example"] = "Value Example"
  map1["red"] = "FF0000"
  fmt.Println(map1)
  
  // Initialize a map with literal values
  map2 := map[int]bool{
    4: false, 
    6: false,
    42: true,
  }
  
  //Access individual elements using the key
  fmt.Println(map1["Red"])
  fmt.Println(map2[42])
  
  // Use range to iterate through maps
  for key, value := range map2{
    fmt.Printf("%d: %t\n", key, value)
  }
}
```

### Channel

channels 用来在进程中通信，它是先进先出first-in,first-out(FIFO)`队列。每个channel 只能支持一种数据类型。channels默认阻塞，但是可以使用select声明做成非阻塞。跟slices 和 maps 一样，channels必须用make()方法初始化才能够使用。

channel使用例子：

```go
package main
import (
  "log"
  "time"
)

// Do some processing that takes a long time 
// in a separate thread and signal when done
func process(doneChannel chan bool){
  time.Sleep(time.Second * 3)
  doneChannel <- true
}

func main(){
  // Each channel can support one data type
  // Can also use custom types
  var doneChannel chan bool
  
  // Channels are nil until initialized with make
  doneChannel = make(chan bool)
  
  // Kick of a lengthy process that will signal when complete
  go process(doneChannel)
  
  // Get the first bool available in the channel
  // This is a blocking operation so execution
  // will not progress until value is received
  tempBool := <-doneChannel
  log.Println(tempBool)
  // or to simply ignore the value but still wait 
  // <-doneChannel
  
  // Start another process thread to run in background
  // and signal when done
  go process(doneChannel)
  
  // Make channel non-blocking with select statement
  // This gives you the ability to continue executing
  // even if no message is waiting in the channel
  var readyToExit = false
  for !readyToExit {
    select {
      case done := <-doneChannel:
      	log.Println("Done message received.", done)
      	readyToExit = true
      default:
      log.Println("No done signal yet. Waiting.")
      time.Sleep(time.Millisecond * 500)
    }
  }
}
```



## 流程控制(Control structures)

跟其他语言类似，Go也提供了用于控制程序流程的语句，其中最常用的是`if`, `for`loops, `switch`。同时Go也提供`goto`语句，但是不提倡使用。

#### if

if 声明可以结合 if, else if , else 来使用。Go的if除了提供常规语言有的功能，还可以在判断条件前面加入表达式，并且可以创建能够作用于整个if表达式的临时变量。

```go
package main
import (
  "fmt"
  "math/rand"
)

func main(){
  x := rand.Int()
  
  if x < 100 {
    fmt.Println("x is less than 100.")
  }
  
  if x < 1000 {
    fmt.Println("x is less than 1000")
  } else if x < 10000 {
    fmt.Println("x is less than 10,000")
  } else {
    fmt.Println("x is greater than 10,000")
  }
  
  fmt.Println("x:", x)
  
  // You can put a statement before the condition
  // The variable scope of n is limited
  if n := rand.Int();n > 1000 {
    fmt.Println("n is greater than 1000.")
    fmt.Println("n:", n)
  } else {
    fmt.Println("n is not greater than 1000.")
    fmt.Println("n:", n)
  }
  // n is no longer available past the if statement
  
}
```

#### for

for 使用与C和java中的类似，但是循环没有while这个关键字。

```go
package main
import "fmt"
func main(){
  // Basic for loop
  for i := 0; i < 3;i++{
    fmt.Println("i:", i)
  }
  
  // For used as a while loop
  n := 5
  for n < 10 {
    fmt.Println(n)
    n++
  }
}
```

#### range

`range`关键字常用于遍历`slice`,`map`,以及其他数据结构，它常与`for`关键字同用。`range`返回key, value变量。

```go
package main
import "fmt"
func main(){
  intSlice := []int{2,4,6,8}
  for key, value := range intSlice {
    fmt.Println(key, value)
  }
  myMap := map[string]string{
    "d": "Donut",
    "o": "Operator",
  }
  
  // Iterate over a map
  for key, value := range myMap {
    fmt.Println(key, value)
  }
  
  // Iterate but only utilize keys
  for key := range myMap {
    fmt.Println(key)
  }
  
  // Use underscore to ignore keys
  for _, value := range myMap {
    fmt.Println(value)
  }
}
```

#### switch, case , fallthrough, and default

`switch`关键字可以根据变量状态来选择执行的分支，与C语言的switch分支类似。

默认没有`fallthrough`功能，这样一旦匹配一个分支并完成执行代码就会完全退出。而没有匹配到任何一个分支就会执行`default`语句。

与if语句类似，你可以在需要switch的变量之前添加表达式，创建的变量作用域限定为整个switch语句。

switch 例子：

```go
package main
import (
	"fmt"
  "math/rand"
)

func main(){
  x := 42
  
  switch x {
    case 25:
    	fmt.Println("X is 25")
    case 42:
    	fmt.Println("X is the magical 42")
      // Fallthrough will continue to next case
    	fallthrough
  	case 100:
   	 	fmt.Println("X is 100")
    case 1000:
    	fmt.Println("X is 1000")
    default:
      fmt.Println("X is something else.")
  }
  
  // Like the if statement a statement
  // can be put in front of the switched variable
  switch r := rand.Int(); r {
    case r % 2 :
    fmt.Println("Random number r is even.")
    default:
    fmt.Println("Random number r is odd.")
  }
  // r is no longer available after the switch statement
}
```

#### goto

`goto`作为保留字很少被使用。

```go
package main
import "fmt"
func main(){
  goto customLabel
  
  // Will never get executed because 
  // the goto statement will jump right past this line
  fmt.Println("Hello")
  customLabel:
  fmt.Println("World")
}
```



## Defer

用defer 修饰函数，它会在当前函数退出时运行。这种方式能够确保一个函数在程序退出之前被执行，这样会在清理现场、关闭文件方面很有用。

常用场景是关闭文件或者数据库链接。

需要注意的是，defer 的函数是在被包裹的函数退出时候运行，如果你将一个defer 调用放到 for 循环中，它并不会在每次循环结束运行。

```go
package main
import (
	"log"
  "os"
)

func main(){
  file, err := os.Create("test.txt")
  if err != nil {
    log.Fatal("Error creating file.")
  }
  defer file.Close()
  // It is important to defer after checking the errors
  // You can't call Close() on a nil object
  // if the open failed
  // ...perform some other actions here
  
  // file.Close() will be called before final exit
}
```



## 包(Package)

包就是文件夹目录，每个目录就是自己的包。创建子文件夹就是在创建一个新的包。不含有子目录会导致结构没有层次感。子目录会帮助组织项目代码结构。

包名字应该跟目录名字匹配，或者直接叫main。当定义一个main包说明它不会被引用到其他应用当中，而是作为编译运行的程序。包使用`import`关键字引入。

引入单个包

```go
import "fmt"
```

引入多个包

```go
import (
	"fmt"
  "log"
)
```



## 类(Classes)

Go 理论上没有类，但是与面向对象语言相比也仅仅很少的细微差别。

从概念上来讲，我们可以把Go语言认为是面向对象语言，虽然它仅包含一些面向对象的基本特性。虽然没有继承和多态，但是我们可以使用类型嵌套和接口来代替。

我们可以将定义好的类型加上与之关联操作的函数看做是类，而这个类型的实例就可以看作是对象。



#### Inheritance 继承

Go语言中可以使用类型嵌套模拟继承功能

```go
package main
import (
	"fmt"
  "reflect"
)

type Person struct {
  Name string
  Age int
}

type Doctor struct {
  Person Person
  Specialization string
}

func main(){
  nanodano := Person{
    Name : "NanoDano",
    Age : 29,
  }
  
  drDano := Doctor {
    Person: nanodano,
    Specialization: "Hacking",
  }
  
  fmt.Println(reflect.TypeOf(nanodano))
  fmt.Println(nanodano)
  fmt.Println(reflect.TypeOf(drDano))
  fmt.Println(drDano)
}
```

#### Polymorphism 多态

Go没有多态，你可以创建通用接口，以供给多种类型实现。接口定义了一个或多个方法声明，这些类型必须满足这些声明才能与接口兼容。

#### Coustructors 构造器

Go 语言也没有构造器，但是可以通过New()函数来实现工厂初始化。

同样的Go语言也没有解构函数，垃圾回收机制会自动销毁对象。如果确实有需要手动处理的可以使用`defer`来在当前函数返回之前来调用处理。

```go
package main
import "fmt"
type Person struct {
  Name string
}

func NewPerson() Person {
  return Person{
    Name: "Anonymous",
  }
}

func main() {
  p := NewPerson()
  fmt.Println(p)
}
```

#### Methods 方法

方法本质上就是属于特殊类型的函数，使用点号调用

```go
myObject.myMethod()
```

方法的通用形式是对某一种类型所有操作的一系列函数，所有这些相关函数都有相同的第一个参数，这个参数就是所操作的数据。

Go语言取消了将对象作为参数传递到函数中的方法，取而代之是在定义方法的时候将对象放入特定位置。这个位置被称为接收者(receiver)，使用小括号包裹，置于函数名之前。这个接收者可以是type 也可以是type的指针。

```go
package main
import "fmt"
type Person struct {
  Name string
}

// Person function receiver
func (p Person) PrintInfo() {
  fmt.Printf("Name: %s\n", p.Name)
}

// Person pointer receiver
// If you did not use the pointer receivers
// it would not modify the person object
// Try removing the asterisk here and seeing how the program changes behavior
func (p *Person) ChangeName(newName string){
  p.Name = newName
}

func main(){
  nanodano := Person{Name: "NanoDano"}
  nanodano.PrintInfo()
  nanodano.ChangeName("Just Dano")
  nanodano.PrintInfo()
}
```

#### Operator overloading

Go没有操作符重载，所以我们不能够通过`+`来相加两个结构，但是你可以在结构类型上定义Add()函数来实现类似`dataSet1.Add(dataSet2)`操作。

## Goroutines

Goroutines是语言内置的轻量级线程，通过在函数前面加`go`关键字，使函数能够单独在线程中执行。

Go提供 mutexes，在官方文档https://golang.org/pkg/sync中可以查看。一般进程间通信我们使用Channels来共享和通信。

下面的例子`log`包线程安全，而`fmt`不是

```go
package main
import (
	"log"
  "time"
)

func countDown() {
  for i := 5; i >= 0; i--{
    log.Println(i)
    time.Sleep(time.Millisecond * 500)
  }
}

func main(){
  // Kick off a thread
  go countDown()
  
  // Since functions are first-class
  // you can write an anonymous function for a goroutine
  go func() {
    time.Sleep(time.Second * 2)
    log.Println("Delayed greetings!")
  }()
  
  // Use channels to signal when complete
  // Or in this case just wait
  time.Sleep(time.Second * 4)
}
```



## 帮助文档(Documentation)

### 线上文档

官网：https://golang.org/

文档：https://golang.org/doc/

标准库: https://golang.org/pkg/

### 线下文档

Go 提供`godoc`命令，它能够起一个离线的官方文档

```go
# Get fmt package information
godoc fmt

# Get source code for fmt package 
godoc -src fmt

# Get specific function infomation
godoc fmt Printf

# Get source code for function
godoc -src fmt Printf

# Run HTTP server to view HTML documentation
godoc -http=localhost:9999
```



