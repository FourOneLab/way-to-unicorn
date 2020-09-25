
# 恐慌、恢复和延迟执行

## 运行时恐慌（panic）

> 这种异常只有在程序运行时才会抛出。

```go
package main

func main() {
	slice := []int{1, 2, 3}
	print(slice[3])

    // output：

	// panic: runtime error: index out of range

	// goroutine 1 [running]:                           ID为1的用户级线程在此panic被引发时正在运行
	// main.main()                                      用户级线程包装的go函数，在这里是main函数，所以这个是主用户级线程
	// demo.go:5 +0x3d
	// panic 被引起时正在执行的代码                         +0x3d 表示此行代码对于其所属的入口程序计数偏移量
	// Process finished with exit code 2                Go语言中因panic而退出的程序一般退出码都是2
}
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
	"errors"
	"fmt"
)

func main() {
	fmt.Println("Enter function main.")
	// 引发 panic
	panic(errors.New("something wrong")) // 手动调用panic引发运行时恐慌
	p := recover()
	fmt.Printf("panic: %s\n", p) // 手动调用recover恢复panic
	fmt.Println("Exit function main.")
}
```

recover函数的调用没有任何作用，甚至都没有机会执行，因为panic一旦发生，控制权就会迅速地沿着调用栈的反方向传播，所以在panic函数调用之后的代码，根本就没有执行的机会。

**如果我们在调用recover函数时没有发生panic，那么该函数不会做任何事情，并且只会返回一个nil**。正确调用recover函数需要配合defer语句一起使用。

## defer语句

> [Golang 规范](https://golang.org/ref/spec#Defer_statements)所描述：延迟函数（deferred functions）在所在函数返回前，以与声明相反的顺序立即被调用。

defer语句是被用来延迟执行代码的，延迟到该语句所在的函数即将执行结束的那一刻，无论结束执行的原因是什么。有一些调用表达式不能出现在defer语句中：

- 针对Go语言**内建函数**的调用表达式
- 针对**unsafe包**中的函数的调用表达式

在defer语句中，被调用的函数可以是有名称的，也可以是匿名的，把这里的函数叫做defer函数或者延迟函数，**注意，被延迟执行的是defer函数，而不是defer语句**。

无论函数结束执行的原因是什么，其中defer函数调用都会在它即将结束执行的那一刻执行，即使执行结束的原因是一个panic也会是这样。因此，需要连用defer语句和recover函数，才能恢复一个已经发生的panic。

```go
package main

import (
	"errors"
	"fmt"
)

func main() {
	fmt.Println("Enter function main.")
	defer func() {
		fmt.Println("Enter defer function.")
		// p!=nil 判断确实发生了panic
		if p := recover(); p != nil {
			fmt.Printf("panic: %s\n", p)
		}
		fmt.Println("；Exit defer function.")
	}()
	// 引发 panic
	panic(errors.New("something wrong"))
	fmt.Println("Exit function main.")
}
```

**注意defer语句的位置，尽量写在函数体的开始处，因为在引发panic的语句之后的所有语句，都没有任何执行的机会**。

### defer函数的执行顺序

在一个函数中，defer函数调用的执行顺序与它们分别所属的defer语句的出现顺序（执行顺序）完全相反。当一个函数即将结束执行时，其中写在最下面的defer函数调用会最先执行，其次是写在它上边、与它距离最近的defer函数调用，以此类推，最上边的defer函数调用最后一个执行。

如果一个for语句中包含一个defer语句，那么defer语句的执行次数，取决于循环的迭代次数，并且同一个defer语句被执行一次，其中的defer函数调用就会产生一次，而且这些函数调用同样不会被立即执行。

在defer语句每次被执行的时候，Go语言会把它携带的defer函数以及其参数值另行存储到一个**后进先出的栈**中。

需要执行某个函数中的defer函数调用的时候，Go语言会先拿到对应的栈，然后从该栈中一个一个地取出defer函数及其参数值，并逐个执行调用。

### defer内部实现

Go 运行时（runtime）使用一个链表来实现 LIFO。实际上，一个 defer 结构体持有一个指向下一个要被执行的 defer 结构体的指针：

```go
type _defer struct {
   siz     int32
   started bool
   sp      uintptr
   pc      uintptr
   fn      *funcval
   _panic  *_panic
   link    *_defer // 下一个要被执行的延迟函数
```

当一个新的 defer 方法被创建的时候，它被附加到当前的 Goroutine 上，然后之前的 defer 方法作为下一个要执行的函数被链接到新创建的方法上（可以理解为从链表的表头开始插入新的defer语句中的函数）：

```go
func newdefer(siz int32) *_defer {
   var d *_defer
   gp := getg() // 获取当前 goroutine
   [...]
   // 延迟列表现在被附加到新的 _defer 结构体
   d.link = gp._defer
   gp._defer = d // 新的结构现在是第一个被调用的
   return d
}
```

现在，后续调用会从栈的顶部依次出栈延迟函数：

```go
func deferreturn(arg0 uintptr) {
	gp := getg() // 获取当前 goroutine
	d:= gp._defer // 拷贝延迟函数到一个变量上
	if d == nil { // 如果不存在延迟函数就直接返回
		return
	}
	[...]
	fn := d.fn // 获取要调用的函数
	d.fn = nil // 重置函数
	gp._defer = d.link // 把下一个 _defer 结构体依附到 Goroutine 上
	freedefer(d) // 释放 _defer 结构体
	jmpdefer(fn, uintptr(unsafe.Pointer(&arg0))) // 调用该函数
}
```

如上，并没有循环地去调用延迟函数，而是一个接一个地出栈。

### defer与return

> return之后的语句先执行，defer后的语句后执行。

延迟函数访问返回的结果的唯一方法是使用**命名返回参数**。因为，只要声明函数的命名返回参数后，就会在函数初始化时赋予该类型的零值，而且在函数体作用域可见。

> 如果延迟函数是一个**匿名函数**，并且所在函数存在命名返回参数，同时该命名返回参数在匿名函数的作用域中，匿名函数会在返回参数返回前访问并修改它们。
>
> 如果延迟函数是一个**命名函数**，那么命名返回参数就需要以变量的形式传入，Go是值传递，所以defer函数中修改的不是命名返回参数本身了，只有传入命名返回参数的指针类型，才能在defer函数中修改return的返回值。

```go
func main() {
   fmt.Printf("with named param, x: %d\n", namedParam())
   fmt.Printf("without named param, x: %d\n", notNamedParam())
}
func namedParam() (x int) {	// 命名返回参数
   x = 1					// 命名返回参数在匿名函数的作用域中
   defer func() { x = 2 }()	// 匿名延迟函数
   return x
}

func notNamedParam() (int) {
   x := 1
   defer func() { x = 2 }()	// 匿名延迟函数
   return x
}
with named param, x: 2
without named param, x: 1
```

### defer与panic

可以将defer函数和recover函数混合使用。defer 最大的功能是 panic 后依然有效，所以defer可以保证一些资源一定会被关闭，从而避免一些异常出现的问题。

> recover 函数是一个用于重新获取对恐慌（panicking）goroutine 控制的内置函数。**recover函数仅在延迟函数内部时才有效**。

_defer 结构体链接了一个_panic 属性，该属性在 panic 调用期间被链接：

```go
func gopanic(e interface{}) {
   [...]
   var p _panic
   [...]
   d := gp._defer // 当前附加的 defer 函数
   [...]
   d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))
   [...]
}
```

#### 没有recover函数的情况

```go
package main

import (
    "fmt"
)

func main() {
    defer_call()

    fmt.Println("main 正常结束")
}

func defer_call() {
    defer func() { fmt.Println("defer: panic 之前1") }()
    defer func() { fmt.Println("defer: panic 之前2") }()

    panic("异常内容")  //触发defer出栈，但此时没有执行panic函数

    defer func() { fmt.Println("defer: panic 之后，永远执行不到") }()
}
// output
defer: panic 之前2
defer: panic 之前1
panic: 异常内容
//... 异常堆栈信息
```

#### 使用recover函数的情况

```go
import (
    "fmt"
)

func main() {
    defer_call()

    fmt.Println("main 正常结束")
}

func defer_call() {

    defer func() {   // 使用recover捕获panic，并恢复异常
        fmt.Println("defer: panic 之前1, 捕获异常")
        if err := recover(); err != nil {
            fmt.Println(err)
        }
    }()

    defer func() { fmt.Println("defer: panic 之前2, 不捕获") }()  // 没有捕获panic，正常执行defer函数

    panic("异常内容")  //触发defer出栈，但此时不执行panic函数

    defer func() { fmt.Println("defer: panic 之后, 永远执行不到") }()
}
// output
defer: panic 之前2, 不捕获
defer: panic 之前1, 捕获异常
异常内容
main 正常结束
```

如果在recover一个panic的时候，函数需要返回一些error信息，那么还是要使用命名返回参数。

```go
func main() {
   fmt.Printf("error from err1: %v\n", err1())
   fmt.Printf("error from err2: %v\n", err2())
}

func err1() error {  // 匿名返回参数
   var err error

   defer func() {
      if r := recover(); r != nil {
         err = errors.New("recovered")
      }
   }()
   panic(`foo`)

   return err  // return语句并未执行
}

func err2() (err error) {	// 命名返回参数
   defer func() {	         // recover在defer内部
      if r := recover(); r != nil {
         err = errors.New("recovered")
      }
   }()

   panic(`foo`)

   return err
}
// output
error from err1: <nil>
error from err2: recovered
```

### defer中包含panic

```go
package main

import (
    "fmt"
)

func main()  {

    defer func() {   // 后出栈
       if err := recover(); err != nil{   // 捕获第二个panic
           fmt.Println(err)
       }else {
           fmt.Println("fatal")
       }
    }()

    defer func() {   // 先出栈
        panic("defer panic")  // panic覆盖main中的那个panic
    }()

    panic("panic")   // 触发defer出栈，但此时不执行panic函数
}
// output
defer panic
```

panic仅有最后一个可以被revover捕获。

### defer函数中包含子函数

```go
package main

import "fmt"

func function(index int, value int) int {

    fmt.Println(index)

    return index
}

func main() {
    defer function(1, function(3, 0))  // 第一个入栈，此时执行子函数，输出3，后出栈，后输出1
    defer function(2, function(4, 0))  // 第二个入栈，此时执行子函数，输出4，先出栈，先输出2
}
```

### defer练习题

```go
package main

import "fmt"

func main() {
	fmt.Println(DeferFunc1(1)) // 第一个执行
	fmt.Println(DeferFunc2(1)) // 第二个执行
	fmt.Println(DeferFunc3(1)) // 第三个执行
	DeferFunc4()               // 第四个执行
}
func DeferFunc1(i int) (t int) { // 1、i=1，传入
	t = i // 2、此时 t=1
	defer func() {
		t += 3 // 4、此时t=1+3，函数返回结果4
	}()
	return t // 3、此时t=1
}

func DeferFunc2(i int) int {  // 1、i=1，传入
	t := i // 2、此时t=1
	defer func() {
		t += 3   // 4、此时，t=1+3，与匿名返回参数无关
	}()
	return t // 3、此时t=1，可以理解为将t的值赋予给匿名返回参数，函数返回结果1
}

func DeferFunc3(i int) (t int) { // 1、i=1，传入
	defer func() {
		t += i // 3、此时t=2+1，函数返回结果3
	}()
	return 2 // 2、将2赋予给命名返回参数t，此时t=2
}

func DeferFunc4() (t int) {
	defer func(i int) {
		fmt.Println(i) // 2、i的值是由t赋予的，此时i=0，在控制台输出0
		fmt.Println(t) // 5、此时t=2，在控制台输出2
	}(t) // 1、将命名返回参数作为形参传入给defer语句中的匿名函数，此时t=0（在函数调用时初始化为int的零值）
	t = 1 // 3、此时t=1
	return 2 // 4、将2赋予给命名返回参数，此时t=2
}
// output
4
1
3
0
2
```
