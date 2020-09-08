---
title: 16-panic&recover&defer
date: 2019-11-25T11:15:47.522182+08:00
draft: false
---

## 运行时恐慌（panic）

> 这种异常只有在程序运行时才会抛出。

```go
slice := []int{1,2,3}
print(slice[3])

// panic: runtime error: index out of range

// goroutine 1 [running]:       ID为1的用户级线程在此panic被引发时正在运行
// main.main()                  用户级线程包装的go函数，在这里是main函数，所以这个是主用户级线程
//  /Users/haolin/GeekTime/Golang_Puzzlers/src/puzzlers/article19/q0/demo47.go:5 +0x3d
// painc被引起时正在执行的代码     +0x3d 表示此行代码对于其所属的入口程序计数偏移量
// exit status 2        Go语言中因panic而退出的程序一般退出码都是2
```

Go语言运行时系统，会在执行到这个代码的时候抛出一个索引越界的异常，当panic被抛出后，没有在程序里添加任何保护措施的话，程序（或者代表它的进程）就会在打印出panic的详细信之后终止运行。

上述例子抛出的异常是runtime包中抛出的panic，在这个panic中，包含一个`runtime.Error`接口类型的值，`runtime.Error`接口内嵌了error接口，并做了一点扩展。在panic右边的内容，就是这个panic包含的`runtime.Error`类型值的字符串表示形式。

## 从panic被引发到程序终止运行的大致过程

1. 函数中的某行代码有意或无意地引发了一个panic
2. 初始的panic详情会被建立起来
3. 该程序的控制权会立即从此代码转移至调用起所属函数的那个代码行上（也就是调用栈中的上一级）
4. 此行代码所属函数的执行随即被终止
5. 控制权立即转移至再上一级的调用代码处（控制权一级一级沿着调用栈的反方向传播至顶端，我们编写的最外层函数）
6. 对于其他用户级线程，最外层函数是go函数，对于主用户级线程，最外层函数是main函数
7. 控制权最终被Go运行时系统收回
8. 随后程序崩溃并终止运行，承载程序这次运行的进程会随之死亡并消失

在这个控制权传播的过程中，panic详情会被积累和完善，并会在程序终止之前打印出来。

> Go语言的内建函数panic是专门用于引发panic的，panic函数使程序开发者可以在程序运行期间报告异常。这与从函数返回错误值的意义是完全不同的，当我们的函数返回一个非nil的错误值时，函数的调用方有权选择不处理，并且不处理的后果往往是不致命的（即不至于程序僵死或崩溃）。

当一个panic发生时，如果不施加任何保护措施，那么导致的直接后果就是程序崩溃，这是致命的。

**panic只能在程序运行期间抛出的程序异常，它的详情会在控制权传播的过程中，被逐渐地积累和完善，并且，控制权会一级一级地沿着调用栈的反方向传播至顶端**。

## 让panic包含一个值

- 如果一个panic是无意间引发的，那么其中的值只能由Go语言运行时系统给定
- 如果使用panic函数有意地引发一个panic，那么可以自行指定其包含的值

在调用panic函数时，把某个值作为参数传递给该函数，即可自行指定其包含的值。由于panic函数的唯一一个参数是空接口（interface{}）类型，从语法上，可以接收任何类型的值。**但是，最好传入error类型的错误值，或者其他可被有效序列化（有效序列化指的是更易读，且易于表示形式转换）的值**。

在`fmt`包中的各种打印函数来说，error类型的值Error方法与其他类型的String方法是等价的，它们的唯一结果都是String类型。通过占位符`%s`输出它们的字符串表示形式。一旦程序异常，就应该把异常信息记录到程序日志中。

所以如果某个值可能会被记录到日志中，就应该给它关联String方法，如果是error类型，就应该关联Error方法。

## 对panic施加保护避免程序崩溃

Go语言内建的recover专用于恢复panic，平息运行时恐慌。recover函数无需任何参数，并且会返回一个空接口类型的值。**如果用法正确**，这个值实际上就是即将恢复的panic包含的值。如果这个panic是我们调用panic函数而引发的，那么该值同时会是我们此次调用panic函数时传入的参数值副本。

```go
package main

import (
 "fmt"
 "errors"
)

func main() {
 fmt.Println("Enter function main.")
 // 引发 panic。
 panic(errors.New("something wrong"))       // 手动调用panic引发运行时恐慌
 p := recover()
 fmt.Printf("panic: %s\n", p)               // 手动调用recover恢复panic
 fmt.Println("Exit function main.")
}
```

recover函数的调用没有任何作用，甚至都没有机会执行，因为panic一旦发生，控制权就会迅速地沿着调用栈的反方向传播，所以在panic函数调用之后的代码，根本就没有执行的机会。

**如果我们在调用recover函数时没有发生panic，那么该函数不会做任何事情，并且只会返回一个nil**。正确调用recover函数需要配合defer语句一起使用。

### defer语句

defer语句是被用来延迟执行代码的，延迟到该语句所在的函数即将执行结束的那一刻，无论结束执行的原因是什么。有一些调用表达式不能出现在defer语句中：

- 针对Go语言内建函数的调用表达式
- 针对unsafe包中的函数的调用表达式

在defer语句中，被调用的函数可以是有名称的，也可以是匿名的，把这里的函数叫做defer函数或者延迟函数，**注意，被延迟执行的是defer函数，而不是defer语句**。

无论函数结束执行的原因是什么，其中defer函数调用都会在它即将结束执行的那一刻执行，即使执行结束的原因是一个panic也会是这样。因此，需要连用defer语句和recover函数，才能恢复一个已经发生的panic。

```go
package main

import (
 "fmt"
 "errors"
)

func main() {
 fmt.Println("Enter function main.")
 defer func(){
  fmt.Println("Enter defer function.")
  // p!=nil 判断确实发生了panic
  if p := recover(); p != nil {
   fmt.Printf("panic: %s\n", p)
  }
  fmt.Println("Exit defer function.")
 }()
 // 引发 panic。
 panic(errors.New("something wrong"))
 fmt.Println("Exit function main.")
}
```

**注意defer语句的位置，尽量写在函数体的开始处，因为在引发panic的语句之后的所有语句，都没有任何执行的机会**。

## 一个函数中有多个defer语句，defer函数的执行顺序

在一个函数中，defer函数调用的执行顺序与它们分别所属的defer语句的出现顺序（执行顺序）完全相反。当一个函数即将结束执行时，其中写在最下面的defer函数调用会最先执行，其次是写在它上边、与它距离最近的defer函数调用，以此类推，最上边的defer函数调用最后一个执行。

> 如果一个for语句中包含一个defer语句，那么defer语句的执行次数，取决于循环的迭代次数，并且同一个defer语句被执行一次，其中的defer函数调用就会产生一次，而且这些函数调用同样不会被立即执行。

### defer语句执行时发生的事情

在defer语句每次被执行的时候，Go语言会把它携带的defer函数以及其参数值另行存储到一个先进先出队列中。这个队列与defer语句所属的函数是对应的，相当于一个栈。

需要执行某个函数中的defer函数调用的时候，Go语言会先拿到对应的队列，然后从该队列中一个一个地取出defer函数及其参数值，并逐个执行调用。
