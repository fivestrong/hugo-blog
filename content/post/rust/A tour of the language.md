---
title: "A Tour of the Language"
date: 2020-01-03T09:36:09+08:00
tags: ["rust"]
categories: ["rust"]
draft: true
---

## 原生类型

Rust 包含以下原生类型:

- bool: 布尔类型包含 true or false

- char: 字符类型，表示单个字符，比如'e'

- Integer: 由多个bit 宽度表示的整数，Rust 最大支持128 bits.

  | signed | unsigned |
  | :----: | :------: |
  |   I8   |    u8    |
  |  i16   |   u16    |
  |  I32   |   u32    |
  |  i64   |   u64    |
  |  i128  |   u128   |

- isize: 自适应类型,根据CPU类型决定的整型, 32-bit CPU表示i32, 64-bit CPU 表示i64

- usize: 同上，根据CPU类型表示的无符号整型

- f32: 32位浮点数整型

- f64: 64位浮点数整型

- [T;N]  固定大小的数组，T表示泛型(代表任意类型)；N表示数组大小

- [T] 动态大小数组，T代表类型

- str: 字符串切片， 主要作为引用使用， &str

- (T, U, ..): 序列，类似python中的元组，T，U代表不同类型

- fn(i32) -> i32: 函数类型，接收i32类型，返回i32类型

## 变量定义

Rust 使用 *let* 定义变量，默认是不可变变量。在变量之前使用 *mut* 关键字可以定义为可变变量

```rust
// variables.rs
fn main(){
  let target = "world";// 不可变变量
  let mut greeting = "Hello";// 可变变量
  println!("{}, {}", greeting, target);
  greeting = "How do you do?";
  target = "mate"; // 这一行报错"cannot assign twice to immutable variable"
  println!("{}, {}", greeting, target);
}
```



## 函数

```rust
// functions.rs
fn add(a: u64, b: u64) -> u64 {
  a + b
}
fn main(){
  let a: u64 = 17;//显示声明类型
  let b = 3;// 类型推导
  let result = add(a, b);
  println!("Result {}", result);
}
```

使用 *fn* 定义函数，函数名为 *add* ，这个函数接收两个参数 a，b 类型都是u64， 函数主体写在在{}内，函数使用 *->* 表示返回值，该函数返回u64，省略表示函数没有返回值；这个add 函数也有类型，fn(u64, u64) -> u64 所以函数可以赋值给变量并可以作为参数传到其他函数中。

函数add 没有使用 *return* 来返回值，因为在Rust中最后一个表达式将被自动返回。当然我们也可以使用 *return* 来提前返回。

如果不显示声明函数返回值，函数默认返回一个值，( ) Unit 类型，类似于C/C++中的 *void* 。

## 闭包

Rust 支持闭包，闭包与函数类似，但是在定义闭包时候包含了更多的环境变量和作用域变量信息。闭包定义没有像函数那样的函数名，但是它可以被分配给变量。

最简单的闭包定义: let my_closure = | | ( ); 这个闭包没有参数，没有任何卵用。可以类似函数那样的调用这个闭包，my_closure()。

```rust
// closures.rs
fn main(){
  let doubler = |x| x * 2;//||中包含参数，参数可以有类型，|x: u32|;闭包可以存入变量;闭包body可以为单一表达式
  let value = 5;
  let twice = doubler(value);//根据变量名调用闭包
  println!("{} doubled is {}", value, twice);
  
  let big_closure = |b, c| {
    let z = b + c;
    z * twice
  }; //多行表达式
  let some_number = big_closure(1,2);
  println!("Result from closure: {}", some_number);
}
```

闭包的主要作用是作为高阶函数的参数。比如标准库中 *thread:;spawn* 接收一个闭包，这个闭包包含你需要在其他线程中运行的代码。

## 字符串

在 Rust 中字符串有两种类型： &str type(读作 stir) ；String type。Rust strings 是UTF-8编码的 byte 序列。

```rust
// strings.rs
fn main(){
  let question = "How are you ?";  // a &str type
  let person: String = "Bob".to_string(); // String type
  let hello = String::from("你好"); // unicodes "你好", String type
  
  println!("{}!{}{}", hello, question, person);
}
```

Strings 被声明到heap(堆)上，&str 被指向已存在的string，这个string可能在heap(堆)，stack(栈)，编译后的目标代码的数据段中。

## 条件判断

### if else 判断语句
条件语句的语法与其他语言类似，都是 *if {} else {}* 结构

````rust
// if_else.rs
fn main(){
  let rust_is_awesome = true;
  if rust_is_awesome {
    println!("Indeed");
  } else {
    println!("Well, you should try Rust!");
  } // 这里不需要;
}
````

在Rust中，if结构不是语句(statement)，而是表达式(expression)。在通常的语言中，语句(statements)不返回值，而表达式(expression)返回值。这个差别使得Rust中的 *if else* 总是返回一个值。这个值可能使一个空的 ( ) unit type 或者一个其他实际的值。任何在{}中的最后一行都会成为 *if else* 的返回值，并且需要注意的是 *if* *else* 分支的返回值类型必须相同。



```rust
// if_assign.rs
fn main(){
  let result = if 1==2 {
    "Wait, what ?"
  } else {
    "Rust makes sense"
  }; //这里需要有一个分号，因为let statement 需要分号结尾。
  println!("You know what ? {}.", result);
}
```

Rust的条件判断可以赋值给变量，如上所述。上面的代码如果删除else语句块，编译器会报错。因为如果if条件为假，返回值为( )，这个条件判断会有两个可能的返回值类型( ) 或者 &str。Rust 不允许在一个变量中存储多个类型的值。



```rust
// if_else_no_value.rs
fn main(){
  let result = if 1==2 {
    "Nothing makes sense";//加了分号，编译器会忽略这一行直接返回()
  } else {
    "Sanity reigns";
  };
  println!("Result of computation: {:?}", result);
}
```



###  Match 表达式

Rust有类似于C语言中的switch语句，即match expressions



````rust
// match_expression.rs
fn req_status() -> u32 {
  200
}
fn main(){
  let status = req_status();
  match status {
    200 => println!("Success"), // match arms, 每个分支必须返回相同类型，这里返回()
    404 => println!("Not Found"),
    other => { //这里匹配所有其他结果(catch all),如果不打算用这些匹配结果，可以使用_抛弃结果。
      println!("Request failed with code: {}", other);
    }
  }//match 跟if else一样，也可以使用let赋值给变量，这里需要分号结尾。
}
````



## 循环语句

Rust的循环语句有三个结构 *loop* *while* *for* ，这些结构都可以使用 *continue* *break* 关键字控制循环。 


### loop
````rust
// loops.rs
fn main(){
  let mut x = 1024;
  loop {
    if x < 0 {
      break;
    }
    println!("{} more runs to go", x);
    x -= 1;
  }
}
````

loop 代表无限循环， 由条件x < 0 控制 break 跳出。可以给loop加一个标签，多层循环可以根据标签直接跳出最外层。



```rust
// loop_labels.rs
fn silly_sub(a: i32, b: i32) -> i32 {
  let mut result = 0;
  // 注意！这里标签使用的使' 
  'increment: loop {
    if result == a {
      let mut dec = b;
      'decrement: loop {
        if dec == 0 {
          // 直接跳出'increment循环
          break 'increment;
        } else {
          result -= 1;
          dec -= 1
        }
      }
    } else {
      result += 1;
    }
  }
  result
}

fn main(){
  let a = 10;
  let b = 4;
  let result = silly_sub(a, b);
  println!("{} minus {} is {}", a, b, result);
}
```


### while
while 循环示例

```rust
// while.rs
fn main(){
  let mut x = 1000;
  while x > 0 {
    println!("{} more runs to go", x);
    x -= 1;
  }
}
```


### for
for 语句只能用于可以被转化为迭代器(iterators)的类型。比如说Range类型。

```rust
// for_loops.rs
fn main(){
  // does not include 10
  print!("Normal ranges: ");
  for i in 0..10 {
    print!("{},", i);
  }
  
  println!();
  print!("Inclusive ranges: ");
  // counts till 10
  for i in 0..=10{
    print!("{},", i);
  }
}
```



## 用户自定义类型

用户自定义类型可以由三种方式自定义：

- structs	
- enums
- unions

### 结构体

结构体定义

```rust
// unit_struct.rs
struct Dummy; //unit struce
fn main(){
  let value = Dummy; //value为Dummy的实例，并且为零值。
}
```



```rust
// tuple_struct.rs
struct Color(u8, u8, u8);
fn main(){
  let white = Color(255,255,255);
  // 通过索引获取值
  let red = white.0;
  let green = white.1;
  let blue = white.2;
  
  println!("Red value: {}", red);
  println!("Green value: {}", green);
  println!("Blue value: {}\n", blue);
  
  let orange = Color(255,165,0);
  
  // 解构获取值
  let Color(r, g, b) = orange;
  println!("R: {}, G: {}, B: {} (orange)", r, g, b);
  // 忽略某个值
  let Color(r, _, b) = orange;
}
```



当数据少于4、5个的时候，tuple struct 是理想的模型选择。当数据类型字段多于3个，建议采用类C风格的struct

```rust
// stucts.rs
struct Player {
  name: String,
  iq: u8,
  friends: u8,
  score: u16
}

fn bump_player_score(mut player: Player, score: u16){
  player.score += score;
  println!("Updated player stats:");
  println!("Name: {}", player.name);
  println!("IQ: {}", player.iq);
  println!("Friends: {}", player.friends);
  println!("Score: {}", player.score);
}

fn main(){
  let name = "Alice".to_string();
  let player = Player {
    name, // 当结构字段与值字段名称相同，可以省略为一个，es6 也有类似的语法
    iq: 182,
    friends: 133,
    score: 1200
  };
  bump_player_score(player, 120)
}
```



### 枚举类型

enums 可以包含不同种类的结构，比如原生结构，structs, enum等

```rust
// enums.rs
#[derive(Debug)]
enum Direction {
  N, //称做 成员(variants)
  E,
  S,
  W
}
enum PlayerAction {
  Move {
    direction: Direction,
    speed: u8
  },
  Wait,
  Attack(Direction)
}

fn main(){
  let simulated_player_action = PlayerAction::Move{
    direction: Direction::N,
    speed: 2,
  };
  match simulated_player_action {
    PlayerAction::Wait => println!("Player wants to wait"),
    PlayerAction::Move {direction, speed } => {
      println!("Player wants to move in direction {:?} with speed {}", direction, speed)
    }
    PlayerAction::Attack(direction) => {
      println!("Player wants to attack direction {:?}", direction)
    }
  }
}
```



### 作用于类型上的函数和方法