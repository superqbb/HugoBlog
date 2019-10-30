---
title: "Golang语法特点"
date: 2019-10-27T03:19:48+08:00
draft: false
tags: ["golang基础"]
series: ["golang入门"]
categories: ["golang"]
---



### 1、Golang关键字

Golang的语法整体来说比较简洁，风格高度统一。
Golang只有25个关键字

| break  | case  | chan  | const | continue |
| --- | --- | --- | --- | --- |
| default  | defer | else  | fallthrough | for |
| func | go  | goto | if  | import |
| interface  | map  | package  | range  | return |
| select  | struct | switch  | type | var |

语法简单，上手速度快，由于编程风格的统一，因此在代码review，浏览其他开源的golang项目时，会有非常熟悉的感觉。

### 2、自动识别变量类型

``` go
sum := 10
str := "hello word"

//上述语句简化了变量声明与赋值
var sum int
sum = 10

var str string
str = “hello word”
```



### 3、for表达式

```go
package main

import "fmt"

func main(){
         sum := 0
         for index :=0; index < 10 ; index++ {
                 sum += index
         }
         fmt.Println("sum is equal to ", sum)
}
```

Golang中没有while关键字，可以使用for进行代替，忽略expression1和expression3：

```go
sum := 1
for sum < 1000 {
     sum += sum
}
```

### 4、Go里面switch默认相当于每个case最后带有break
Go的switch非常灵活，表达式不必是常量或整数，也可以使用多个逗号分隔的方式匹配多个值，执行的过程从上至下，直到找到匹配项；而如果switch没有表达式，它会匹配true。

```go
i := 10
switch i/10 {
    case 1:
             fmt.Println("i is equal to 1")
    case 2, 3, 4:
             fmt.Println("i is equal to 2, 3 or 4")
    case 10:
             fmt.Println("i is equal to 10")
    default:
             fmt.Println("All I know is that i is an integer")
}
```

### 5、切片slice

### 6、函数定义func

函数 `func`是Go里面的核心设计，它通过关键字 `func` 来声明，它的格式如下：

```go
func funcName( input1 type1, input2 ...type2) (output1 type1, output2 type2) {
         //这里是处理逻辑代码
         //返回多个值
         return value1, value2
}
```

* Go函数支持变参，即入参可以是一个不定参数，可以接受多个不确定数量的参数，比如input2

```go
func test1(args ...string) { //可以接受任意个string参数
        for _, v:= range args{
            fmt.Println(v)
        }
}
func main(){
    var strss= []string{
            "qwr",
            "234",
            "yui",
            "cvbc",
        }
    test1(strss...) //切片被打散传入
}
```

* Go语言比C更先进的特性，其中一点就是函数能够返回多个值
* 如果没有返回值，那么就直接省略最后的返回信息

### 7、defer 延迟语句
Go语言中有种不错的设计，即延迟（defer）语句，你可以在函数中添加多个defer语句。当函数执行到最后时，这些defer语句会按照逆序执行，最后该函数返回。特别是当你在进行一些打开资源的操作时，遇到错误需要提前返回，在返回前你需要关闭相应的资源，不然很容易造成资源泄露等问题。如下代码所示，我们一般写打开一个资源是这样操作的：

```go
func ReadWrite() bool {
         file.Open("file")
         // 做一些工作
         if failureX {
                  file.Close()
                  return false
         }
         if failureY {
                  file.Close()
                  return false
         }
         file.Close()
         return true
}
```

我们看到上面有很多重复的代码，如果代码多的时候，还有可能忘记写file.close()。Go的defer有效解决了这个问题。使用它后，不但代码量减少了很多，而且程序变得更优雅。在defer后指定的函数会在函数退出前调用。

**代码使用defer改写如下：**

``` go
func ReadWrite() bool {
        defer fmt.println("all done")
         file.Open("file")
         //open之后，可以马上写一个defer，函数退出前即可把文件关闭
         defer file.Close()
         if failureX {
                  return false
         }
         if failureY {
                  return false
         }
         defer fmt.println("1st defer")
         return true
}
```

值得注意的是，defer执行顺序是先进后出。上面的例子，在函数退出之前，会先打印"1st defer"，再执行file.Close()，最后打印"all done"。

### 8、Panic和Recover
一般来说，我们应该尽可能地使用错误，而不是使用 panic 和 recover
一般如果函数执行的异常，会抛出一个error类型的返回参数，我们在函数外部对error进行处理即可。

``` go

func foo(param int)(ret int, err error)
{
    //若函数正常执行，返回的err为nil，否则返回一个不为nill的error
    ...  
}

//调用时的代码
func main(){
    n, err := foo(0)
    if err != nil {
        //  错误处理
    } else {
        // 使用返回值n
    }
}
```

但有时候，一些函数不会返回error，而是直接panic或者运行时的panic。Go没有像Java那样的异常机制(try catch)，它不能抛出异常，而是使用了`panic`和`recover`机制。**Panic的时候如果程序不使用recover进行异常捕获处理，会导致整个程序退出。**

panic 有两个合理的用例。

* 1、发生了一个不能恢复的错误，此时程序不能继续运行。 一个例子就是 web 服务器无法绑定所要求的端口。在这种情况下，就应该使用 panic，因为如果不能绑定端口，啥也做不了。
* 2、发生了一个编程上的错误。 假如我们有一个接收指针参数的方法，而其他人使用 nil 作为参数调用了它。在这种情况下，我们可以使用 panic，因为这是一个编程错误：用 nil 参数调用了一个只能接收合法指针的方法。

8.1 主动panic
下面这个函数演示了如何在过程中使用panic

```go
var user = os.Getenv("USER")
//值得一提的是，golang import一个包后，如果包内有init函数，会自动执行init函数，对包内的资源做一些初始化的工作
func init() {
         if user == "" {
                 //主动使用panic
                 panic("no value for $USER")
         }
}
```

> 顺带一提：main函数引入包初始化流程图
![p48-2358709.png](/images/p48-2358709.png)

8.2 当函数发生 panic 时，它会终止运行，在执行完所有的延迟函数后，程序控制返回到该函数的调用方。

```go
package main

import (  
    "fmt"
)

func fullName(firstName *string, lastName *string) {  
    defer fmt.Println("deferred call in fullName")
    if firstName == nil {
        panic("runtime error: first name cannot be nil")
    }
    if lastName == nil {
        panic("runtime error: last name cannot be nil")
    }
    fmt.Printf("%s %s\n", *firstName, *lastName)
    fmt.Println("returned normally from fullName")
}

func main() {  
    defer fmt.Println("deferred call in main")
    firstName := "Elon"
    fullName(&firstName, nil)
    fmt.Println("returned normally from main")
}

```

> 该函数会打印：
> deferred call in fullName  
> deferred call in main  
> panic: runtime error: last name cannot be nil
> goroutine 1 [running]:  
> main.fullName(0x1042bf90, 0x0)  
>     /tmp/sandbox060731990/main.go:13 +0x280
> main.main()  
>     /tmp/sandbox060731990/main.go:22 +0xc0

8.3 `recover` 是一个内建函数，用于重新获得 `panic` 协程的控制。
值得注意的是：**只有在延迟函数的内部，调用 recover 才有用。**
在延迟函数内调用 recover，可以取到 panic 的错误信息，并且停止 panic 续发事件（Panicking Sequence），程序运行恢复正常。如果在延迟函数的外部调用 recover，就不能停止 panic 续发事件。

下面这个函数检查作为其参数的函数在执行时是否会产生panic：

```go
func throwsPanic(f func()) (b bool) {
         defer func() {
                 if x := recover(); x != nil {
                          b = true
                 }
         }()
         f() //执行函数f，如果f中出现了panic，那么就可以恢复回来
         return
}
```

### 9、go语言中面向对象编程
下面的例子演示了golang中定义对象，定义public、private属性，对象函数的方法

```go
package main

import "fmt"

type Human struct {
     name string
     age int
     phone string
}

type Student struct {
     Human //匿名字段
     school string
}

type Employee struct {
     Human //匿名字段
     company string
}

//Human定义method
func (h *Human) SayHi() {
     fmt.Printf("Hi, I am %s you can call me on %s\n", h.name, h.phone)
}



//Employee的method重写Human的method
func (e *Employee) SayHi() {
     fmt.Printf("Hi, I am %s, I work at %s. Call me on %s\n", e.name,
     e.company, e.phone) //Yes you can split into 2 lines here.
}

func main() {
     mark := Student{Human{"Mark", 25, "222-222-YYYY"}, "MIT"}
     sam := Employee{Human{"Sam", 45, "111-888-XXXX"}, "Golang Inc"}
     mark.SayHi()
     sam.SayHi()
}
```

Go里面的面向对象是如此的简单，没有任何的私有、公有关键字，通过大小写来实现(大写开头的为公有，小写开头的为私有)，方法也同样适用这个原则。

### 10、非侵入式接口
编译时对“引用”的类和接口定义的依赖，我们称之为“侵入性”的；任何显式的“接口”、“基类”都是侵入性的，不可避免的带来编译期依赖；即使这些依赖很小，但依然有办法而且应该尽可能消除。

我们先来看看java中的实现接口

```java
public interface Geometry{
  public float Area();
}

public class Rect implements Geometry {
   ...
   @override
   public float Area(){
     //具体实现
     ...
   }
   
}
```





Go语言的接口设计是一种非侵入性的设计，作为服务提供者不用预先知道所需要实现的接口，也不用为了实现某一个接口而import一个专门的包。而接口可以被定义在客户端，按需求定义，同时接口与接口之间也可以相互赋值，这样就不用过多忧心接口的粒度，给接口设计带来了很大的灵活性。

**animal.go**

```go
package pkg

import "fmt"

type Animal interface{
	Eat()
}

type Bird interface {
	Fly()
}

//定义Dog时，我们并没有在代码的任何地方告诉Dog或者Pig这两个struct它们需要去实现Animal接口
type Dog struct {
	Weight float64
}

//Dog实现了Eat接口，即认为是Animal
func(d *Dog) Eat(){
	fmt.Printf("体重%.1fkg的狗正在进食...\n",d.Weight)
}

type Pig struct {
	Weight float64
}

//Pig实现了Fly接口，即认为是Bird
func(d *Pig) Fly(){
	fmt.Printf("体重%.1fkg的猪正在飞...\n",d.Weight)
}

//Pig实现了Fly接口，即认为是Animal
func(d *Pig) Eat(){
	fmt.Printf("体重%.1fkg的猪正在进食...\n",d.Weight)
}
```

**main.go**

```go
package main

import (
	"fmt"
	"github.com/superqbb/GoCourse/Object/pkg"
)

func Dinner(a pkg.Animal){
	a.Eat()
}

func WindCome(b pkg.Bird){
	b.Fly()
}

//非侵入式接口演示
func main(){
	myDog := &pkg.Dog{Weight:4}
	myPig := &pkg.Pig{Weight:80}

	//因为Dog,Pig都实现了Animal接口，编译器会自动识别
	Dinner(myDog)
	Dinner(myPig)

	fmt.Printf("风来了...\n")
	WindCome(myPig)
}
```

[SourceCode]: https://github.com/superqbb/GoCourse/tree/master/Object	"SourceCode"

