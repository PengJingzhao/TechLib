

# Golang 基础语法介绍

## 1. 基础类型和内置函数

### 基础类型

Go 语言提供了一套丰富的内建类型，包括：

### 1.1 值类型

- • **布尔类型**：`bool`，表示逻辑上的真（`true`）或假（`false`）。
- • **整型**：包括有符号和无符号整数，如`int8`、`int16`、`int32`、`int64`、`uint8`（别名`byte`）、`uint16`、`uint32`、`uint64`、`rune`（类似于`int32`）、平台相关的`int`和`uint`以及`uintptr`（无符号整型，用于存放一个指针）。
- • **浮点型**：`float32`、`float64`、`complex64` （32 位实数和虚数）、`complex128` （64 位实数和虚数）用于表示有小数部分的数字。
- • **字符串**：`string`，表示字符的序列，Go 中的字符串是不可变的。

### 1.2 派生类型:

- • **指针类型（pointer）**。

```
type a unsafe.Pointer
```

- • **数组类型**。

```
type a [8]int
```

-**结构化类型（struct）**。

```
type a struct {}
```

- • **Channel 类型**。

```
type a chan int
```

- • **函数类型**。

```
type fn func()
```

- • **切片类型（slice）**。

```
type a []int
```

- • **接口类型（interface）**。

```
type inter interface {}
```

- • **字典类型（map）**。

```
type a map[int]int
```

## 2. 变量、常量、nil 与零值、方法、包、可见性、指针

### 2.1 变量声明

1. \1. 使用var关键字声明，且需要注意的是，与大多数强类型语言不同，Go语言的声明变量类型位于变量名称的后面。Go语句结束不需要分号。

```
var a int
var aStr  string
```

1. \1. 使用 := 赋值

```
a := 1 
```

等效

```
var a = 1 
```

等效

```
var a  int = 1 
```

### 2.2 常量声明

使用 const 来声明一个常量，一个常量在声明后不可改变。

```
const a  = 1
```

等效

```
const a int = 1
```

### 2.3 nil和零值

只声明未赋值的变量，其值为nil。类似于java中的“null”。

```
var a map[int]int // a是nil
```

没有明确初始值的变量声明会被赋予它们的零值。零值是：

- • 数值类型为 0，
- • 布尔类型为 false，
- • 字符串为 ""（空字符串）。

### 2.4 方法和包

使用func关键字来定义一个方法，后面跟方法名，然后是参数，返回值（如果有的话，没有返回值则不写）。使用package 引入对应包。package main 是主包入口,调用func main() 运行。

```
package main

import (
    "fmt"
)

func main() {
    fmt.Println("hello word")
}
```

自定义包(main包可引用demo包来打印demo函数)

```
package demo

func demo() int {
    return 1
}
```

#### 多返回值

Go 函数与其他编程语言一大不同之处在于支持多返回值，这在处理程序出错的时候非常有用。有点类似python。

```
func demo(a,b int) int,int {
    return a,b
}
```

#### 变长参数

```
func demo(a ...int) {
    for _,v := range a {
        fmt.Println(v)
    }
}
```

#### 可见性

跟其他语言的public、private、protected不同。go是根据首字母大小写区分。首字母大写相当于public，可以在包外直接访问这些变量、属性、函数和方法。首字母小写只能在包内访问，因此 Go 语言类属性和成员方法的可见性都是包一级的，而不是类一级的。

#### 指针

```
package main

import "fmt"

func  main() {
    a := 0
    fmt.Println(&a) // 使用&取a的内存地址
}
```

## 3. 流程控制

Go 支持多种流程控制语句，包括条件语句和循环语句。

### 条件语句

- • **if**：用于基于条件执行不同的代码块。
- • **if-else**：提供条件为真或假时执行的代码块。
- • **if-else if-else**：提供多个条件分支。
- • **switch**：类似于其他语言中的`switch`或`case`语句，但 Go 的`switch`更灵活，每个case自带break（跳出当前case）。

```
if x > 10 {
    fmt.Println("x is greater than 10")
} else {
    fmt.Println("x is not greater than 10")
}

switch os := runtime.GOOS; os {
case "darwin":
    fmt.Println("OS X.")
case "linux":
    fmt.Println("Linux.")
default:
    // freebsd, openbsd,
    // plan9, windows...
    fmt.Printf("%s.\n", os)
}
```

### 循环语句

- • **for**：Go 中唯一的循环结构，非常灵活，可以模拟`while`和`for`循环。
- • **range**：用于遍历数组、切片、字符串、映射或通道（channel）。

```
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

for _, value := range mySlice {
    fmt.Println(value)
}
```

### goto

goto 调整执行位置

```
package main

import (
    "fmt"
)

func main() {
    i := 0
Start:
    i++
    if i < 10 {
        goto Start
    }
    goto End
End:
    fmt.Println(i)
}
```

等效

```
package main

import (
    "fmt"
)

func main() {
    i := 0
    for i = 0; i < 10; i++ {

    }
    fmt.Println(i)
}
```

## 4. 函数、方法、面向对象

### 函数

如前所述，函数是 Go 中封装代码块的方式。

### 方法

在 Go 中，方法是一种特殊的函数，它“绑定”到类型上。通过类型的方法，你可以为那个类型定义一组操作。

```
type Circle struct {
    radius float64
}

// Circle的Area方法
func (c Circle) Area() float64 {
    return math.Pi * c.radius * c.radius
}
```

### 面向对象

虽然 Go 不是纯面向对象的语言，但它通过结构体和接口支持面向对象的编程范式。结构体定义了复合数据类型，而接口定义了对象的行为。

```
package main

import "fmt"

type animal interface {
    GetName() string
}

type Cat struct {
}

func (c *Cat) GetName() string {
    return "cat"
}

type Dog struct {
}

func (c *Dog) GetName() string {
    return "dog"
}

func GetAnimalName(an animal) string {
    return an.GetName()
}

func main() {

    animalName := GetAnimalName(&Dog{})
    fmt.Println(animalName)
}
```

## 5. 并发编程

Go 语言因其强大的并发支持而著名，主要通过 goroutines 和 channels 实现。

### Goroutine

Goroutine 是 Go 的运行时（runtime）中的轻量级线程。启动 goroutine 很简单，只需在函数调用前加上`go`关键字。

```
go sayHello("Alice")
```

### Channel

Channel 是 goroutines 之间的通信机制。你可以通过 channel 发送和接收值，实现 goroutines 之间的同步和通信。

```
ch := make(chan string)
go func() {
    ch <- "Hello from goroutine"
}()

msg := <-ch
fmt.Println(msg)
```

以上便是 Golang 基础语法的简要介绍，包括基础类型、函数、流程控制、函数与方法、面向对象以及并发编程。希望这能为你学习 Go 语言提供一个良好的起点。