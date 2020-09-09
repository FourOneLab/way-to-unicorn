---
title: 19-context
date: 2019-11-25T11:15:47.526182+08:00
draft: false
---

- [0.1. 使用Context包实现一对多goroutine协作流程](#01-使用context包实现一对多goroutine协作流程)
  - [0.1.1. 撤销信号](#011-撤销信号)
  - [0.1.2. 撤销信号在上下文树中的传播](#012-撤销信号在上下文树中的传播)
  - [0.1.3. 通过Context携带数据，并获取数据](#013-通过context携带数据并获取数据)
- [0.2. 总结](#02-总结)

使用WaitGroup可以实现一对多的goroutine协作流程同步，如果一开始不能确定子任务的goroutine数量，那么使用WaitGroup值来协调它们和分发子任务的goroutine就存在一定的风险。

> 一个解决方案是：分批地启用执行子任务的goroutine。

## 0.1. 使用Context包实现一对多goroutine协作流程

```go
func coordinateWithContext() {
 total := 12
 var num int32
 fmt.Printf("The number: %d [with context.Context]\n", num)
//  调用context.Background和context.WithCancel创建一个可撤销的context对象ctx和一个撤销函数cancelFunc
 ctx, cancelFunc := context.WithCancel(context.Background())
 for i := 1; i <= total; i++ {
    //  每次迭代创建一个新的goroutine
  go addNum(&num, i, func() {
    //   在子goroutine中原子性的Load num变量
   if atomic.LoadInt32(&num) == int32(total) {
    // 如果num与total相等，表示所有子goroutine执行完成
    // 调用context的撤销函数
    cancelFunc()
   }
  })
 }
// 调用Done函数，并试图针对该函数返回的通道进行接收操作
// 一旦cancelFunc被调用，针对该通道的接收操作就会马上结束
 <-cxt.Done()
 fmt.Println("End.")
}
```

`context.Context`类型在1.7版本引入后，许多标准库都进行了扩展支持，包括：`os/exec`，`net`,`database/sql`,`runtime/pprof`，`runtime/trace`。

**context类型是一种非常通用的同步工具，它的值不但可以被任意地扩散，而且还可以被用来传递额外的信息和信号。更具体的说，Context类型可以提供一类代表上下文的值，此类值是并发安全的，可以被传播到多个goroutine**。

- Context类型是一个接口类型，context包中实现该接口的所有的私有类型，都是基于某个数据类型的指针类型，所以如此传播并不会影响该类型值的功能和安全。
- Context类型的值是可以衍生，可以通过Context值产生出任意个子值，这些子值可以携带其父值的属性和数据，也可以响应通过其父值传达的信号。

> Context值共同构成了一棵代表了上下文全貌的树形结构。这棵树的树根（上下文根节点）是一个已经在context包中预定义好的Context值，它是全局唯一的。**通过调用context.Background函数可以获取它**。此处的上下文根节点只是最基本的支点，不通过任何额外的功能，既不能被撤销也不能携带任何数据。

context包中包含四个用于衍生context值的函数：

- `WithCancel`：产生一个可撤销的parent的子值
- `WithDeadline`,`WithTimeout`：产生一个会定时撤销的parent的子值
- `WithValue`：产生一个会携带额外数据的parent的子值

这些函数的第一个参数类型都是`context.Context`，名称都是parent，这个位置上的参数都是它们将产生的Context值的父值。

### 0.1.1. 撤销信号

Context接口类型中有两个方法与撤销相关：

1. Done方法返回一个元素类型为`struct{}`的接收通道，这个通道的用途不是传递元素值，而是让调用方去感知撤销当前Context值的那个信号。一旦当前Context值被撤销，接收通道会立即关闭，（对于一个未包含任何元素值的通道，它的关闭使任何针对它的接收操作立即结束）。
2. Err方法，让Context值的使用方感知到撤销信号的同时得到撤销的具体原因，该方法的结果是error类型的，并且其值只可能等于：
   1. `context.Canceled`变量的值：表示手动撤销
   2. 或者`context.DeadlineExceeded`变量的值：表示给定的过期时间已到而导致撤销

`context.WithCancel`函数产生一个可撤销的Context值，还会获得一个用于出发撤销信号的函数，通过调用该函数，可以触发针对这个Context值的撤销信号，一旦触发，撤销信号会立即被传达给这个Context值，并由它的Done方法的结果值（一个接收通道）表达出来。

> 撤销函数只负责触发信号，对应的可撤销的Context值也只负责传达信号，它们不会管后边具体的撤销操作，代码在感知到撤销信号后，可以进行任意的操作，Context值对此并没有任何的约束。

更进一步，这里的撤销最原始的含义是:

- 终止程序对某种请求（比如HTTP请求）的响应，
- 取消对某个指令（如SQL指令）的处理，

这是创建Context包和Context类型时的**初衷**。

### 0.1.2. 撤销信号在上下文树中的传播

Context包中包含四个用于衍生Context值的函数，其中的`WithCancel`，`WithDeadline`，`WithTimeout`都是被用来基于给定的Context值产生可撤销的子值。

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
// 返回一个可被撤销的context值和一个出触发撤销信号的函数
```

撤销函数被调用后，Context值会先关闭它内部的接收通道，即Done方法会返回的那个通道。然后，它会向它的所有子值（或者说子节点）传达撤销信号。这些子值会继续把撤销信号传播下去，最后这个context值会断开它与其父值之间的关联。

![images](/images/context.png)

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

`WithDeadline`和`WithTimeout`函数生成的Context值也是可撤销的，它们不但可以被手动撤销，还会依据在生成时被给定的过期时间，自动地进行定时撤销（定时撤销的功能借助内部的计时器来实现），撤销的同时释放内部的计时器。

**注意，通过调用`context.WithValue`函数得到的Context值是不可撤销的。撤销信号在被传播时，如遇到它们则会直接跨过，并试图将信号直接传给它们的子值**。

### 0.1.3. 通过Context携带数据，并获取数据

```go
func WithValue(parent Context, key, val interface{}) Context
```

WithValue函数在产生新的包含数据的Context值的时候，需要三个参数，即：父值、键（与map对键的约束类似，类型必须可判等）、值。因为从中获取数据的时候，根据给定的键来查找对应的值。不过这里并没有用map来存储数据。

Context类型的Value方法就是被用来获取数据的，在调用含数据的Context值的Value方法时，它会先判断给定的键，是否与当前值中存储的键相等，如果相等就把该值中存储的值直接返回，否则就到其父值中继续查找。如果父值中仍未存储相等的键，那么继续向上直到查找到根节点。

**除了包含数据的Context可以存储数据，其他的Context值都不能携带数据**，Context的Value方法在向上查找的过程中会直接跳过这几种类型的Context值。

> 如果调用的Value方法所属的Context本身就不包含数据，那么实际调用的就会是其父值的Value方法。因为这几种Context值的实际类型是结构体，它们通过将父值嵌入到自身来表达父子关系。

Context接口并没有提供改变数据的方法，因此在通常情况下，只能通过上下文树中添加含数据的Context值来存储新的数据，或者通过撤销此种值的父值丢弃相应的数据。**如果存储在这里的数据可以从外部改变，那么必须自行保证安全**。

## 0.2. 总结

Context类型的实际值分为三种：

1. 根Context值
2. 可撤销的Context值
   1. 手动撤销，手动调用撤销函数
   2. 定时撤销，设置定时撤销的时间，且不可更改，可在过期时间到达之前手动进行撤销
3. 含数据的Context值，可以携带数据，每个值可以存储一对键值对，调用Value方法，它会沿着树根的方向逐个值进行查找，如果发现相等立即返回，否则将在最后返回nil

所有的Context值共同构成一颗上下文树，这棵树的作用域是全局的，根Context值是全局唯一的，不提供任何额外的功能。

**撤销操作是Context值能够协调多个goroutine的关键**，撤销信号总是会沿着上下文树叶子节点的方向传播。

**含数据的Context不能被撤销，能被撤销的Context无法携带数据**，它们共同组成一个整体（上下文树）。
