# 基本并发原语（Mutex）

> 一旦数据被多个线程共享，很可能会产生争用和冲突的情况，这种情况称为**竞态条件（race condition）**，这往往会破坏共享数据的一致性。

**共享数据的一致性代表着：多个线程对共享数据的操作总是可以达到它们各自预期的效果**。

同步的用途有两个：

1. 避免多个线程，在同一时刻操作同一个**数据块**
2. 协调多个线程，避免它们在同一时刻执行同一个**代码块**

这些数据块和代码块的背后都隐含着一种或多种资源（如存储资源，计算资源、I/O资源、网络资源等），把它们看成是共享资源，**同步其实就是在控制多个线程对共享资源的访问**。

## Mutex

常见并发访问问题：

- 多个 goroutine 并发更新同一个资源，像计数器；
- 同时更新用户的账户信息；
- 秒杀系统；
- 往同一个 buffer 中并发写入数据等。

如果没有互斥控制，就会出现一些异常情况，比如计数器的计数不准确、用户的账户可能出现透支、秒杀系统出现超卖、buffer 中的数据混乱，等等，后果都很严重。

### 互斥锁的实现机制

互斥锁（排他锁）是并发控制的一个基本手段，是为了避免竞争而建立的一种并发控制机制。在并发编程中，如果程序中的一部分会被并发访问或修改，那么，为了避免并发访问导致的意想不到的结果，这部分程序需要被保护起来，这部分被保护起来的程序，就叫做临界区。

临界区就是一个被共享的资源，或者说是一个整体的一组共享资源，比如对数据库的访问、对某一个共享数据结构的操作、对一个 I/O 设备的使用、对一个连接池中的连接的调用，等等。

同一个共享资源的相关临界区：

1. 它们可以是一个内含了共享数据的结构体及其方法，
2. 也可以是操作同一块共享数据的多个函数。

使用互斥锁，限定临界区只能同时由一个线程持有。当临界区由一个线程持有的时候，其它线程如果想进入这个临界区，就会返回失败，或者是等待。直到持有的线程退出临界区，这些等待线程中的某一个才有机会接着持有这个临界区。

Mutex（mutual exclusion） 是使用最广泛的同步原语（Synchronization primitives），关于同步原语，可以把它看作解决并发问题的一个基础的数据结构。

同步原语的适用场景：

- 共享资源。并发地读写共享资源，会出现数据竞争（data race）的问题，所以需要 Mutex、RWMutex 这样的并发原语来保护。
- 任务编排。需要 goroutine 按照一定的规律执行，而 goroutine 之间有相互等待或者依赖的顺序关系，我们常常使用 WaitGroup 或者 Channel 来实现。
- 消息传递。信息交流以及不同的 goroutine 之间的线程安全的数据交流，常常使用 Channel 来实现。

### Mutex使用

在 Go 的标准库中，package sync 提供了锁相关的一系列同步原语，这个 package 还定义了一个 Locker 的接口，Mutex 就实现了这个接口。

Locker 的接口定义了锁同步原语的方法集：

```go
type Locker interface {
    Lock()
    Unlock()
}
```

Go 定义的锁接口的方法集很简单，就是请求锁（Lock）和释放锁（Unlock）这两个方法，秉承了 Go 语言一贯的简洁风格。但是，这个接口在实际项目应用得不多，一般会直接使用具体的同步原语，而不是通过接口。

临界区总是需要受到保护，否则就会产生竞态条件，施加保护的重要手段之一，就是**使用实现了某种同步机制的工具**，称为同步原语。

![临界区](/images/sync.png)

一个互斥锁可以被用来包含一个临界区或者一组相关临界区，通过它来保证在同一时刻只有一个goroutine处于该临界区内。每当有goroutine想进入临界区时，都需要先对它进行锁定，离开时要及时进行解锁。

- 锁定操作可以通过调用互斥锁的Lock方法实现
- 解锁操作可以通过调用互斥锁的Unlock方法实现

```go
func(m *Mutex)Lock()
func(m *Mutex)Unlock()
```

**当一个 goroutine 通过调用 Lock 方法获得了这个锁的拥有权后，其它请求锁的 goroutine 就会阻塞在 Lock 方法的调用上，直到锁被释放并且自己获取到了这个锁的拥有权。**

```go
import (
        "fmt"
        "sync"
    )

func main() {
    var count = 0
    // 使用WaitGroup等待10个goroutine完成
    var wg sync.WaitGroup
    wg.Add(10)
    for i := 0; i < 10; i++ {
        go func() {
            defer wg.Done()
            // 对变量count执行10万次加1
            for j := 0; j < 100000; j++ {
                count++
            }
        }()
    }
    // 等待10个goroutine完成
    wg.Wait()
    fmt.Println(count)
}
// 期望的结果：10 * 100000 = 1000000 (一百万)
// 实际的输出：637887 或者 641893 或者 1000000
```

从上面可以看到，每次运行的结果都不一样，因为count++不是一个原子操作，它至少包含几个步骤，比如读取变量 count 的当前值，对这个值加 1，把结果再保存到 count 中。因为不是原子操作，就可能有并发的问题。

```assembly
// count++操作的汇编代码
MOVQ    "".count(SB), AX
LEAQ    1(AX), CX
MOVQ    CX, "".count(SB)
```

上面这个问题还是比较容易发现，但是，很多时候，并发问题隐藏得非常深，即使是有经验的人，也不太容易发现或者 Debug 出来。

针对这个问题，Go 提供了一个检测并发访问共享资源是否有问题的工具：[race detector](https://blog.golang.org/race-detector)，它可以帮助我们自动发现程序有没有 data race 的问题。

> Go race detector 是基于 Google 的 C/C++ [sanitizers](https://github.com/google/sanitizers) 技术实现的，编译器通过探测所有的内存访问，加入代码能监视对这些内存地址的访问（读还是写）。在代码运行的时候，race detector 就能监控到对共享变量的非同步访问，出现 race 的时候，就会打印出警告信息。这个技术在 Google 内部帮了大忙，探测出了 Chromium 等代码的大量并发问题。Go 1.1 中就引入了这种技术，并且一下子就发现了标准库中的 42 个并发问题。现在，race detector 已经成了 Go 持续集成过程中的一部分。

在编译（compile）、测试（test）或者运行（run）Go 代码的时候，加上 race 参数，就有可能发现并发问题。

比如在上面的例子中，加上 race 参数运行，检测一下是不是有并发问题。执行 `go run -race main.go`，就会输出警告信息。

```bash
go run -race main.go
==================
WARNING: DATA RACE
Read at 0x00c0000b0020 by goroutine 8:
  main.main.func1()
      main.go:16 +0x78

Previous write at 0x00c0000b0020 by goroutine 7:
  main.main.func1()
      main.go:16 +0x91

Goroutine 8 (running) created at:
  main.main()
      main.go:13 +0xe4

Goroutine 7 (running) created at:
  main.main()
      main.go:13 +0xe4
==================
376733
Found 1 data race(s)
exit status 66
```

这个警告不但会告诉你有并发问题，而且还会告诉你哪个 goroutine 在哪一行对哪个变量有写操作，同时，哪个 goroutine 在哪一行对哪个变量有读操作，就是这些并发的读写访问，引起了 data race。

> 虽然这个工具使用起来很方便，但是，因为它的实现方式，只能通过真正对实际地址进行读写访问的时候才能探测，所以它并不能在编译的时候发现 data race 的问题。而且，在运行的时候，只有在触发了 data race 之后，才能检测到，如果碰巧没有触发，是检测不出来的。**而且，把开启了 race 的程序部署在线上，还是比较影响性能的。**

运行 `go tool compile -race -S main.go`，通过在编译的时候插入一些指令，在运行时通过这些插入的指令检测并发读写从而发现 data race 问题，就是这个工具的实现机制。

查看计数器例子的代码，重点关注一下 `count++` 前后的编译后的代码：

```bash
0x0063 00099 (main.go:15)       MOVQ    AX, "".j+8(SP)
0x0068 00104 (main.go:16)       MOVQ    "".&count+128(SP), AX
0x0070 00112 (main.go:16)       MOVQ    AX, (SP)
0x0074 00116 (main.go:16)       CALL    runtime.raceread(SB)
0x0079 00121 (main.go:16)       MOVQ    "".&count+128(SP), AX
0x0081 00129 (main.go:16)       MOVQ    (AX), CX
0x0084 00132 (main.go:16)       MOVQ    CX, ""..autotmp_8+16(SP)
0x0089 00137 (main.go:16)       MOVQ    AX, (SP)
0x008d 00141 (main.go:16)       CALL    runtime.racewrite(SB)
0x0092 00146 (main.go:16)       MOVQ    ""..autotmp_8+16(SP), AX
0x0097 00151 (main.go:16)       INCQ    AX
0x009a 00154 (main.go:16)       MOVQ    "".&count+128(SP), CX
0x00a2 00162 (main.go:16)       MOVQ    AX, (CX)
0x00a5 00165 (main.go:15)       MOVQ    "".j+8(SP), AX

```

在编译的代码中，增加了 `runtime.racefuncenter`、`runtime.raceread`、`runtime.racewrite`、`runtime.racefuncexit` 等检测 data race 的方法。通过这些插入的指令，Go race detector 工具就能够成功地检测出 data race 问题了。

要解决data race就可以使用Mutex。这里的共享资源是 count 变量，临界区是 `count++`，只要在临界区前面获取锁，在离开临界区的时候释放锁，就能完美地解决 data race 的问题了。

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    // 互斥锁保护计数器
    var mu sync.Mutex
    // 计数器的值
    var count = 0
    // 辅助变量，用来确认所有的goroutine都完成
    var wg sync.WaitGroup
    wg.Add(10)
    // 启动10个gourontine
    for i := 0; i < 10; i++ {
        go func() {
            defer wg.Done()
            // 累加10万次
            for j := 0; j < 100000; j++ {
                mu.Lock()
                count++
                mu.Unlock()
            }
        }()
    }
    wg.Wait()
    fmt.Println(count)
}
```

> 注意：Mutex 的零值是还没有 goroutine 等待的未加锁的状态，所以不需要额外的初始化，直接声明变量（如 `var mu sync.Mutex`）即可。

很多情况下，**Mutex 会嵌入到其它 struct 中使用**，或者**采用嵌入字段的方式**，直接在struct上调用Lock/Unlock方法。

```go
package main

import (
	"fmt"
	"sync"
)

type Counter struct {
	sync.Mutex
	count int
}

func main() {
	var (
		counter Counter
		wg      sync.WaitGroup
	)

	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < 100000; j++ {
				counter.Lock()
				counter.count++
				counter.Unlock()
			}
		}()
	}
	wg.Wait()
	fmt.Println(counter.count)
}
```

如果嵌入的 struct 有多个字段，我们一般会把 Mutex 放在要控制的字段上面，然后使用空格把字段分隔开来，这是一种很好的代码风格，逻辑会更清晰，也更易于维护。

甚至可以**把获取锁、释放锁、计数加一的逻辑封装成一个方法**，对外不需要暴露锁等逻辑：

```go
package main

import (
	"fmt"
	"sync"
)

// 线程安全的计数器类型
type Counter struct {
	CounterType string
	CounterName string

	mu    sync.Mutex
	count uint64
}

// count变量+1
func (c *Counter) AddOne() {
	c.mu.Lock()
	c.count++
	c.mu.Unlock()
}

// 获取count变量的值
func (c *Counter) GetCount() uint64 {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.count
}

func main() {
	var (
		counter Counter
		wg      sync.WaitGroup
	)

	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			defer wg.Done()
			for j := 0; j < 100000; j++ {
				counter.AddOne()
			}
		}()
	}
	wg.Wait()
	fmt.Println(counter.GetCount())
}

```

#### 注意事项

- 不要重复锁定互斥锁：

> 对于一个已经锁定的互斥锁进行锁定，会立即阻塞当前的goroutine，这个goroutine执行的流程会一直停滞在调用该互斥锁的Lock方法的那行代码上。直到互斥锁的Unlock方法被调用，并且这里的锁定操作成功之后，临界区的代码才会执行。

- 不要忘记解锁互斥锁，必要时使用defer语句

> 可以避免出现重复锁定。因为忘记解锁会使得其他goroutine无法进入到互斥锁保护的临界区中，轻则功能失效，重则死锁崩溃。程序的流程可以分叉也可以被中断，所以一个流程在锁定某个互斥锁之后，紧跟着defer语句进行解锁是比较稳妥的。

- 不要对尚未锁定或者已解锁的互斥锁解锁

> 解锁未锁定的互斥锁会立即引起panic。与死锁的panic一样，无法被恢复。**因此对于每一个锁定操作有且只有一个对应的解锁操作**。

- 不要在多个函数之间直接传递互斥锁

Go语言中的互斥锁是开箱即用的，声明一个`sync.Mutex`类型（该类型是一个**结构体**类型，属于**值类型**）的变量就可以直接使用了。

对于值类型的操作，把它传给一个函数，将它从函数中返回，把它赋给其他变量，让它进入某个通道都会导致它的副本的产生。**原值与副本、副本与副本之间都是完全独立的，它们都是不同的互斥锁**。

> 如果把一个互斥锁作为参数传给了一个函数，那么在这个函数中对传入的锁的所有操作，都不会对存在于该函数之外的那个原锁产生任何的影响。

死锁，当前程序中的主goroutine，以及启用的那些goroutine都已经被阻塞，这些goroutine可以被统称为用户级的goroutine，这就相当于整个程序都已经停滞不前了。

Go语言运行时系统不允许这种情况出现，当发现所以用户级goroutine都处于等待会抛出如下panic：

```go
fatal error: all goroutines are asleep - deadlock!
```

**Go语言运行时系统自行抛出的panic都属于致命错误，无法被恢复，调用recover函数对它们起不到任何作用，程序死锁，必然崩溃**。

当每个互斥锁都只保护一个临界区或者一组相关临界区可以有效避免死锁。

### Mutex实现

Mutex的架构演进：

1. 初代版本（2008）：Mutex 使用一个 flag 来表示锁是否被持有，实现比较简单；
2. 第二阶段（2011）：后来照顾到新来的 goroutine，所以会让新的 goroutine 也尽可能地先获取到锁；
3. 第三阶段（2015）：照顾新来的和被唤醒的 goroutine；
4. 第四阶段（2016）：但是这样会带来饥饿问题，所以目前又加入了饥饿的解决方案。

#### 初版

通过一个 flag 变量，标记当前的锁是否被某个 goroutine 持有。

- 如果这个 flag 的值是 1，就代表锁已经被持有，那么，其它竞争的 goroutine 只能等待；
- 如果这个 flag 的值是 0，就可以通过 CAS（compare-and-swap，或者 compare-and-set）将这个 flag 设置为 1，标识锁被当前的这个 goroutine 持有了。

```go
// CAS操作，当时还没有抽象出atomic包
func cas(val *int32, old, new int32) bool
func semacquire(*int32)
func semrelease(*int32)

// 互斥锁的结构，包含两个字段
type Mutex struct {
    key  int32 // 锁是否被持有的标识
    sema int32 // 信号量专用，用以阻塞/唤醒goroutine
}

// 保证成功在val上增加delta的值
func xadd(val *int32, delta int32) (new int32) {
    for {
        v := *val
        if cas(val, v, v+delta) {
            return v + delta
        }
    }
    panic("unreached")
}

// 请求锁
func (m *Mutex) Lock() {
    if xadd(&m.key, 1) == 1 { // 标识加1，如果等于1，成功获取到锁
        return
    }
    semacquire(&m.sema) // 否则阻塞等待
}

func (m *Mutex) Unlock() {
    if xadd(&m.key, -1) == 0 { // 将标识减去1，如果等于0，则没有其它等待者
        return
    }
    semrelease(&m.sema) // 唤醒其它阻塞的goroutine
}
```

> CAS 指令将给定的值和一个内存地址中的值进行比较，如果它们是同一个值，就使用新值替换内存地址中的值，这个操作是原子性的，保证这个指令总是基于最新的值进行计算，如果同时有其它线程已经修改了这个值，那么，CAS 会返回失败。**CAS 是实现互斥锁和同步原语的基础。**

Mutex 结构体包含两个字段：

- 字段 key：是一个 flag，用来标识这个排他锁是否被某个 goroutine 所持有，如果 key 大于等于 1，说明这个排他锁已经被持有；
  - 0：锁未被持有
  - 1：锁被持有，没有等待者
  - n：锁被持有，有n-1个等待者
- 字段 sema：是个信号量变量，用来控制等待 goroutine 的阻塞休眠和唤醒。

调用 `Lock` 请求锁的时候，通过 `xadd` 方法进行 CAS 操作，`xadd` 方法通过循环执行 CAS 操作直到成功，保证对 key 加 1 的操作成功完成。

- 如果比较幸运，锁没有被别的 goroutine 持有，那么，`Lock` 方法成功地将 key 设置为 1，这个 goroutine 就持有了这个锁；
- 如果锁已经被别的 goroutine 持有了，那么，当前的 goroutine 会把 key 加 1，而且还会调用 `semacquire` 方法，使用信号量将自己休眠，等锁释放的时候，信号量会将它唤醒。

持有锁的 goroutine 调用 `Unlock` 释放锁时，它会将 key 减 1。

- 如果当前没有其它等待这个锁的 goroutine，这个方法就返回了。
- 但是，如果还有等待此锁的其它 goroutine，那么，它会调用 `semrelease` 方法，利用信号量唤醒等待锁的其它 goroutine 中的一个。

初版的 Mutex 利用 CAS 原子操作，对 key 这个标志量进行设置。key 不仅仅标识了锁是否被 goroutine 所持有，还记录了当前持有和等待获取锁的 goroutine 的数量。

> **`Unlock` 方法可以被任意的 goroutine 调用释放锁，即使是没持有这个互斥锁的 goroutine，也可以进行这个操作。这是因为，Mutex 本身并没有包含持有这把锁的 goroutine 的信息，所以，`Unlock` 也不会对此进行检查。Mutex 的这个设计一直保持至今。**

这样会导致一个问题，其它 goroutine 可以强制释放锁，这是一个非常危险的操作，因为在临界区的 goroutine 可能不知道锁已经被释放了，还会继续执行临界区的业务操作，这可能会带来意想不到的结果，因为这个 goroutine 还以为自己持有锁呢，有可能导致 data race 问题。

所以在使用 Mutex 的时候，必须要保证 goroutine 尽可能不去释放自己未持有的锁，一定要遵循“**谁申请，谁释放**”的原则。在真实的实践中，使用互斥锁的时候，很少在一个方法中单独申请锁，而在另外一个方法中单独释放锁，一般都会在同一个方法中获取锁和释放锁。

以前，经常会基于性能的考虑，及时释放掉锁，所以在一些 `if-else` 分支中加上释放锁的代码，代码看起来很臃肿。而且，在重构的时候，也很容易因为误删或者是漏掉而出现死锁的现象。从 1.14 版本起，Go 对 defer 做了优化，采用更有效的内联方式，取代之前的生成 defer 对象到 defer chain 中，defer 对耗时的影响微乎其微了，所以基本上修改成下面简洁的写法也没问题：

```go
// 以前
type Foo struct {
    mu    sync.Mutex
    count int
}

func (f *Foo) Bar() {
    f.mu.Lock()

    if f.count < 1000 {
        f.count += 3
        f.mu.Unlock() // 此处释放锁
        return
    }

    f.count++
    f.mu.Unlock() // 此处释放锁
    return
}

// 修改后
func (f *Foo) Bar() {
    // Lock和Unlock总是成对出现，不多不漏，代码简洁
    f.mu.Lock()
    defer f.mu.Unlock() // 此处释放所锁

    if f.count < 1000 {
        f.count += 3
        return
    }

    f.count++
    return
}

// 如果临界区只是方法中的一部分，为了尽快释放锁，还是应该第一时间调用 Unlock，而不是一直等到方法返回时才释放。
```

初版的 Mutex 实现有一个问题：请求锁的 goroutine 会排队等待获取互斥锁。虽然这貌似很公平，但是从性能上来看，却不是最优的。因为如果能够把锁交给正在占用 CPU 时间片的 goroutine 的话，那就不需要做上下文的切换，在高并发的情况下，可能会有更好的性能。

#### 第二阶段

对 Mutex 做了一次大的调整，调整后的 Mutex 实现如下：

```go
type Mutex struct {
    state int32 // key 修改为 state
    sema  uint32
}

const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken
    mutexWaiterShift = iota
)
```

state 是一个复合型的字段，一个字段包含多个意义，这样可以通过尽可能少的内存来实现互斥锁。所以，state 这一个字段被分成了三部分，代表三个数据：

- muteLocked：持有锁标记，字段的第一位来表示这个锁是否被持有（如果为0表示锁未被持有，大于等于1表示锁已经被持有）
- mutexWoken：唤醒标记，字段的第二位表示是否有唤醒的 goroutine
- mutexWaiters：阻塞等待的waiter数量，剩余的位数代表的是等待此锁的 goroutine 数

请求锁的Lock方法也变复杂了，包括对state字段的位操作和代码逻辑：

```go
func (m *Mutex) Lock() {
    // Fast path: 幸运case，能够直接获取到锁
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        return
    }
    // 第一次获取锁失败时，设置唤醒标识为false
    awoke := false
    for {
        old := m.state  // 保存获取锁失败时state变量的值，此时state不为0，其中包含了三种状态信息
        new := old | mutexLocked // 保证state的新状态中，最后一位（mutexLocked）肯定是1，保证获取到锁的时候，锁处于被持有状态
        if old&mutexLocked != 0 {
            new = old + 1<<mutexWaiterShift //等待者数量加一
        }
        if awoke {
            new &^= mutexWoken   // goroutine是被唤醒的，新状态清除唤醒标志
        }
        if atomic.CompareAndSwapInt32(&m.state, old, new) { // 设置新状态，表示当前goroutine在获取锁的队列中
            if old&mutexLocked == 0 { // 说明原来没有goroutine持有锁，锁已经被释放了，当前goroutine已经持有锁，加锁完成直接跳出for循环并返回
                break
            }
            runtime.Semacquire(&m.sema) // 请求信号量，当前goroutine休眠
            awoke = true    // 从休眠中被唤醒，将awoke设置为true，参与锁的竞争
        }
    }
}
```

先是一个if语句使用CAS检测state字段的`mutexLocked`标志位是否为0，如果没有goroutine持有锁，也没有等待持有锁的goroutine，那么当前的goroutine就直接获得锁。

如果if语句判断为false，那么当前Lock调用就会被阻塞，在锁释放唤醒后，并不是直接获取锁，而是要和正在请求锁的goroutine竞争，**这给后来请求锁的goroutine一个机会，让CPU中正在执行的goroutine有更多机会获取到锁，在一定程度上提高程序的性能。**

具体的过程就在后面的for循环中：

- `old | mutexLocked`的效果：如果old的最后一位是1，那么old不变，否则old的最后一位被设置为1
- `old & mutexLocked`的效果：如果old的最后一位是1，那么新状态中mutexWaiter数量要加1（通过将1向左移动2位实现）
- `new &^= mutexWoken`的效果：先通过muteWoken和与操作获取new的第二位的值，在通过mutexWoken和异或操作将该位置为0

请求锁的 goroutine 有两类，一类是新来请求锁的 goroutine，另一类是被唤醒的等待请求锁的 goroutine。锁的状态也有两种：加锁和未加锁。用一张表格，来说明一下 goroutine 不同来源不同状态下的处理逻辑。

|请求锁的goroutine|当前锁被持有|当前锁未被持有|
|---|---|---|
|新来的|waiter++；休眠|获取到锁|
|被唤醒的|清除mutexWoken标识；重新休眠；加入等待队列|清除mutexWoken标识，获取到锁|

释放锁的Unlock方法也变复杂了：

```go
func (m *Mutex) Unlock() {
    // Fast path: drop lock bit.
    new := atomic.AddInt32(&m.state, -mutexLocked)  // 通过原子操作将state的左右一位置为0，去掉锁标志
    // 如果原来没有加锁，state的值为0
    if (new+mutexLocked)&mutexLocked == 0 { // 对未加锁的排他锁进行解锁，运行时异常
        panic("sync: unlock of unlocked mutex")
    }

    old := new
    for {
        // 如果前半部分为false，说明有等待者
        // 如果后边部分为false，说明还有等待者，但是都没有被唤醒，后面的逻辑就是唤醒其中一个
        // 后半部分为false是另一种情况，此时排他锁又被其他线程持有了
        if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken) != 0 { // 没有等待者，或者有唤醒的waiter，或者锁原来已加锁
            return
        }
        // mutexWaiters减去1，并设置mutexWoken，表示在等待者中唤醒了一个
        new = (old - 1<<mutexWaiterShift) | mutexWoken // 新状态，准备唤醒goroutine，并设置唤醒标志
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            runtime.Semrelease(&m.sema) // 唤醒信号量，唤醒因为这个信号量休眠的goroutine
            return
        }
        old = m.state
    }
}
```

先将加锁置为未加锁的状态，但是这个方法也不能直接返回，还需要一些额外的操作，因为还可能有一些等待这个锁的 goroutine（称之为waiter）需要通过信号量的方式唤醒它们中的一个。所以接下来的逻辑有两种情况。

- 第一种情况，如果没有其它的 waiter，说明对这个锁的竞争的 goroutine 只有一个，那就可以直接返回了；如果这个时候有唤醒的 goroutine，或者是又被别人加了锁，那么，无需我们操劳，其它 goroutine 自己干得都很好，当前的这个 goroutine 就可以放心返回了。
- 第二种情况，如果有等待者，并且没有唤醒的 waiter，那就需要唤醒一个等待的 waiter。在唤醒之前，需要将 waiter 数量减 1，并且将 mutexWoken 标志设置上，这样，Unlock 就可以返回了。

- `old>>mutexWaiterShift == 0`的效果，获取mutexWaiters字段的值，如果为0说明没有等待者
- `old&(mutexLocked|mutexWoken)`的效果，获取mutexLocked和mutexWoken字段的值，如果不为0，说明有已经唤醒的goroutine

**相对于初版的设计，这次的改动主要就是，新来的 goroutine 也有机会先获取到锁，甚至一个 goroutine 可能连续获取到锁，打破了先来先得的逻辑。但是，代码复杂度也显而易见。**

#### 第三阶段

在这里如果新来的 goroutine 或者是被唤醒的 goroutine 首次获取不到锁，它们就会通过自旋（spin，通过循环不断尝试，spin 的逻辑是在runtime 实现的，如下所示）的方式，尝试检查锁是否被释放。在尝试一定的自旋次数后，再执行原来的逻辑。

```go
// Active spinning for sync.Mutex.
//go:linkname sync_runtime_canSpin sync.runtime_canSpin
//go:nosplit
func sync_runtime_canSpin(i int) bool {
    // sync.Mutex是共享的，因此对自旋持保守态度。
    // 自旋仅在多核计算机上运行且GOMAXPROCS>1并且至少有一个其他正在运行的P并且本地runq为空时，才旋转几次。
    // 相对于运行时排他锁，在这里不做被动旋转，因为可以在全局runq或其他P上进行工作。
	if i >= active_spin || ncpu <= 1 || gomaxprocs <= int32(sched.npidle+sched.nmspinning)+1 {
		return false
	}
	if p := getg().m.p.ptr(); !runqempty(p) {
		return false
	}
	return true
}
```

如果可以 spin 的话，第 8 行的 for 循环会重新检查锁是否释放。**对于临界区代码执行非常短的场景来说，这是一个非常好的优化**。因为临界区的代码耗时很短，锁很快就能释放，而抢夺锁的 goroutine 不用通过休眠唤醒方式等待调度，直接 spin 几次，可能就获得了锁。

```go
func (m *Mutex) Lock() {
    // Fast path: 幸运之路，正好获取到锁
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        return
    }
    awoke := false
    iter := 0
    for { // 不管是新来的请求锁的goroutine, 还是被唤醒的goroutine，都不断尝试请求锁
        old := m.state // 先保存当前锁的状态
        new := old | mutexLocked // 新状态设置加锁标志
        if old&mutexLocked != 0 { // 锁还没被释放
            if runtime_canSpin(iter) { // （新增）还可以自旋
                if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                    atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                    awoke = true
                }
                runtime_doSpin()
                iter++
                continue // （新增）自旋，再次尝试请求锁
            }
            new = old + 1<<mutexWaiterShift
        }
        if awoke { // 唤醒状态
            if new&mutexWoken == 0 { // (新增)
                panic("sync: inconsistent mutex state")
            }
            new &^= mutexWoken // 新状态清除唤醒标记
        }
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            if old&mutexLocked == 0 { // 旧状态锁已释放，新状态成功持有了锁，直接返回
                break
            }
            runtime_Semacquire(&m.sema) // 阻塞等待
            awoke = true // 被唤醒
            iter = 0    //（新增）
        }
    }
}
```

#### 第四阶段

经过几次优化，Mutex 的代码越来越复杂，应对高并发争抢锁的场景也更加**公平**。因为新来的 goroutine 也参与竞争，有可能每次都会被新来的 goroutine 抢到获取锁的机会，在极端情况下，等待中的 goroutine 可能会一直获取不到锁，这就是**饥饿问题**。

Mutex 不能容忍这种事情发生。所以增加了饥饿模式，让锁变得更公平，不公平的等待时间限制在 1 毫秒，并且修复了一个大 Bug：总是把唤醒的 goroutine 放在等待队列的尾部，会导致更加不公平的等待时间。

**Mutex 绝不容忍一个 goroutine 被落下，永远没有机会获取锁。不抛弃不放弃是它的宗旨，而且它也尽可能地让等待较长的 goroutine 更有机会获取到锁。**

当前的state字段分为四个部分：

- mutexLocked：第一位，持有锁的标记
- mutexWoken：第二位，唤醒标记
- mutexStarving：第三位，饥饿标记
- mutexWaiters：阻塞等待的waiter数量

```go
type Mutex struct {
    state int32
    sema  uint32
}
const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken
    mutexStarving // 从state字段中分出一个饥饿标记
    mutexWaiterShift = iota
    starvationThresholdNs = 1e6 // 饥饿模式的最大等待时间为1毫秒
    // 一旦等待者等待的时间超过了这个阈值，Mutex 的处理就有可能进入饥饿模式，优先让等待者先获取到锁，新来的同学主动谦让一下，给老同志一些机会。
)
func (m *Mutex) Lock() {
    // Fast path: 幸运之路，一下就获取到了锁
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        return
    }
    // Slow path：缓慢之路，尝试自旋竞争或饥饿状态下饥饿goroutine竞争
    m.lockSlow()
}
func (m *Mutex) lockSlow() {
    var waitStartTime int64
    starving := false // 此goroutine的饥饿标记
    awoke := false // 唤醒标记
    iter := 0 // 自旋次数
    old := m.state // 当前的锁的状态
    for {
        // 锁是非饥饿状态，锁还没被释放，尝试自旋
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            runtime_doSpin()
            iter++
            old = m.state // 再次获取锁的状态，之后会检查是否锁被释放了
            continue
        }
        new := old
        if old&mutexStarving == 0 {
            new |= mutexLocked // 非饥饿状态，加锁
        }
        if old&(mutexLocked|mutexStarving) != 0 {
            new += 1 << mutexWaiterShift // waiter数量加1
        }
        if starving && old&mutexLocked != 0 {
            new |= mutexStarving // 设置饥饿状态
        }
        if awoke {
            if new&mutexWoken == 0 {
                throw("sync: inconsistent mutex state")
            }
            new &^= mutexWoken // 新状态清除唤醒标记
        }
        // 成功设置新状态
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            // 原来锁的状态已释放，并且不是饥饿状态，正常请求到了锁，返回
            if old&(mutexLocked|mutexStarving) == 0 {
                break // locked the mutex with CAS
            }
            // 处理饥饿状态
            // 如果以前就在队列里面，加入到队列头
            queueLifo := waitStartTime != 0
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime()
            }
            // 阻塞等待
            runtime_SemacquireMutex(&m.sema, queueLifo, 1)
            // 唤醒之后检查锁是否应该处于饥饿状态
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
            old = m.state
            // 如果锁已经处于饥饿状态，直接抢到锁，返回
            if old&mutexStarving != 0 {
                if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    throw("sync: inconsistent mutex state")
                }
                // 有点绕，加锁并且将waiter数减1
                delta := int32(mutexLocked - 1<<mutexWaiterShift)
                if !starving || old>>mutexWaiterShift == 1 {
                    delta -= mutexStarving // 最后一个waiter或者已经不饥饿了，清除饥饿标记
                }
                atomic.AddInt32(&m.state, delta)
                break
            }
            awoke = true
            iter = 0
        } else {
            old = m.state
        }
    }
}
func (m *Mutex) Unlock() {
    // Fast path: drop lock bit.
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if new != 0 {
        m.unlockSlow(new)
    }
}
func (m *Mutex) unlockSlow(new int32) {
    if (new+mutexLocked)&mutexLocked == 0 {
        throw("sync: unlock of unlocked mutex")
    }
    if new&mutexStarving == 0 {
        old := new
        for {
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
                return
            }
            new = (old - 1<<mutexWaiterShift) | mutexWoken
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                runtime_Semrelease(&m.sema, false, 1)
                return
            }
            old = m.state
        }
    } else {
        runtime_Semrelease(&m.sema, true, 1)
    }
}
```

跟之前的实现相比，当前的 Mutex 最重要的变化，就是增加饥饿模式。通过加入饥饿模式，可以避免把机会全都留给新来的 goroutine，保证了请求锁的 goroutine 获取锁的公平性，对于我们使用锁的业务代码来说，不会有业务一直等待锁不被处理。

请求锁时调用的 Lock 方法中一开始是 fast path，当前的 goroutine 幸运地获得了锁，没有竞争，直接返回，否则就进入了 lockSlow 方法。这样的设计，方便编译器对 Lock 方法进行内联，在日常程序开发中也可以应用这个技巧。

正常模式下，waiter 都是进入先入先出队列，被唤醒的 waiter 并不会直接持有锁，而是要和新来的 goroutine 进行竞争。**新来的 goroutine 有先天的优势，它们正在 CPU 中运行，可能它们的数量还不少**，所以，在高并发情况下，被唤醒的 waiter 可能比较悲剧地获取不到锁，这时，它会被插入到队列的前面。如果 waiter 获取不到锁的时间超过阈值 1 毫秒，那么，这个 Mutex 就进入到了饥饿模式。

饥饿模式下，Mutex 的拥有者将直接把锁交给队列最前面的 waiter。新来的 goroutine 不会尝试获取锁，即使看起来锁没有被持有，它也不会去抢，也不会 spin，它会乖乖地加入到等待队列的尾部。

如果拥有 Mutex 的 waiter 发现下面两种情况的其中之一，它就会把这个 Mutex 转换成正常模式：

- 此 waiter 已经是队列中的最后一个 waiter 了，没有其它的等待锁的 goroutine 了；
- 此 waiter 的等待时间小于 1 毫秒。

正常模式拥有更好的性能，因为即使有等待抢锁的 waiter，goroutine 也可以连续多次获取到锁。饥饿模式是对公平性和性能的一种平衡，它避免了某些 goroutine 长时间的等待锁。在饥饿模式下，优先对待的是那些一直在等待的 waiter。

### Mutex常见错误使用

#### Lock/Unlock不是成对出现

这会导致要么死锁（缺少Unlock），要么Panic（Unlock一个未加锁的Mutex）。

通常缺少Unlock的常见如下：

1. 代码中有太多的 if-else 分支，可能在某个分支中漏写了 Unlock；
2. 在重构的时候把 Unlock 给删除了；
3. Unlock 误写成了 Lock。

在这种情况下，锁被获取之后，就不会被释放了，这也就意味着，其它的 goroutine 永远都没机会获取到锁。

缺少Lock导致出现对未加锁的Mutex执行Unlock操作，一般都是误操作删除了Lock代码。

#### Copy已使用的Mutex

**Package sync 的同步原语在使用后是不能复制的。**

Mutex 是一个有状态的对象，它的 state 字段记录这个锁的状态。如果复制一个已经加锁的 Mutex 给一个新的变量，那么新的刚初始化的变量居然被加锁了，这显然不符合期望的，因为期望的是一个零值的 Mutex。**关键是在并发环境下，根本不知道要复制的 Mutex 状态是什么，因为要复制的 Mutex 是由其它 goroutine 并发访问的，状态可能总是在变化。**

```go
type Counter struct {
    sync.Mutex
    Count int
}


func main() {
    var c Counter
    c.Lock()
    defer c.Unlock()
    c.Count++
    foo(c) // 复制锁
}

// 这里Counter的参数是通过复制的方式传入的
func foo(c Counter) {
    c.Lock()    // 这个操作阻塞，整个程序处于死锁状态
    defer c.Unlock()
    fmt.Println("in foo")
}

//  运行后的输出如下
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [semacquire]:
sync.runtime_SemacquireMutex(0xc0000200d4, 0xc000062e00, 0x1)
	/opt/go/src/runtime/sema.go:71 +0x47
sync.(*Mutex).lockSlow(0xc0000200d0)
	/opt/go/src/sync/mutex.go:138 +0x105
sync.(*Mutex).Lock(...)
	/opt/go/src/sync/mutex.go:81
main.foo(0x1, 0x1)
	/home/sugoi/go/src/awesomeProject/main.go:23 +0x114
main.main()
	/home/sugoi/go/src/awesomeProject/main.go:18 +0x8a

Process finished with exit code 2
```

Go 在运行时，有死锁的检查机制（[checkdead()](https://golang.org/src/runtime/proc.go?h=checkdead#L4505) 方法），它能够发现死锁的 goroutine。这个例子中因为复制了一个使用了的 Mutex，导致锁无法使用，程序处于死锁的状态。程序运行的时候，死锁检查机制能够发现这种死锁情况并输出错误信息。

可以使用 go vet 工具，把检查写在 Makefile 文件中，在持续集成的时候跑一跑，这样可以及时发现问题，及时修复，而不是在运行时再显示死锁。

```bash
go vet main.go
# command-line-arguments
./main.go:18:6: call of foo copies lock value: command-line-arguments.Counter
./main.go:22:12: foo passes lock by value: command-line-arguments.Counter
```

使用这个工具就可以发现 Mutex 复制的问题，错误信息显示得很清楚，是在调用 foo 函数的时候发生了 lock value 复制的情况，还显示出问题的代码行数以及 copy lock 导致的错误。

检查是通过[copylock](https://github.com/golang/tools/blob/master/go/analysis/passes/copylock/copylock.go)分析器静态分析实现的。这个分析器会分析函数调用、range 遍历、复制、声明、函数返回值等位置，有没有锁的值 copy 的情景，以此来判断有没有问题。可以说，只要是实现了 Locker 接口，就会被分析。

#### 重入

> 可重入锁：当一个线程获取锁时，如果没有其它线程拥有这个锁，那么，这个线程就成功获取到这个锁。之后，如果其它线程再请求这个锁，就会处于阻塞等待的状态。但是，如果拥有这把锁的线程再请求这把锁的话，不会阻塞，而是成功返回，所以叫可重入锁（有时候也叫做递归锁）。只要你拥有这把锁，你可以可着劲儿地调用，比如通过递归实现一些算法，调用者不会阻塞或者死锁。

**Mutex 不是可重入的锁。**因为 Mutex 的实现中没有记录哪个 goroutine 拥有这把锁。理论上，任何 goroutine 都可以随意地 Unlock 这把锁，所以没办法计算重入条件。所以，一旦误用 Mutex 的重入，就会导致报错。

```go
func foo(l sync.Locker) {
    fmt.Println("in foo")
    l.Lock()
    bar(l)
    l.Unlock()
}


func bar(l sync.Locker) {
    l.Lock()
    fmt.Println("in bar")
    l.Unlock()
}


func main() {
    l := &sync.Mutex{}
    foo(l)
}

// 执行输入如下

in foo
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [semacquire]:
sync.runtime_SemacquireMutex(0xc0000a4014, 0x54e000, 0x1)
	/opt/go/src/runtime/sema.go:71 +0x47
sync.(*Mutex).lockSlow(0xc0000a4010)
	/opt/go/src/sync/mutex.go:138 +0x105
sync.(*Mutex).Lock(0xc0000a4010)
	/opt/go/src/sync/mutex.go:81 +0x47
main.bar(0x4dc640, 0xc0000a4010)
	/home/sugoi/go/src/awesomeProject/main.go:16 +0x35
main.foo(0x4dc640, 0xc0000a4010)
	/home/sugoi/go/src/awesomeProject/main.go:11 +0xa5
main.main()
	/home/sugoi/go/src/awesomeProject/main.go:23 +0x3d

Process finished with exit code 2
```

重复的对l进行加锁，导致程序死锁了，因为没有其他goroutine对l进行解锁操作。

自己实现一个重入锁，**关键就是实现的锁要能记住当前是哪个 goroutine 持有这个锁**。

- 方案一：通过 hacker 的方式获取到 goroutine id，记录下获取锁的 goroutine id，它可以实现 Locker 接口。
- 方案二：调用 Lock/Unlock 方法时，由 goroutine 提供一个 token，用来标识它自己，而不是通过 hacker 的方式获取到 goroutine id，但是，这样一来，就不满足 Locker 接口了。

可重入锁（递归锁）解决了代码重入或者递归调用带来的死锁问题，同时它也带来了另一个好处，就是可以要求，只有持有锁的 goroutine 才能 unlock 这个锁。这也很容易实现，因为在上面这两个方案中，都已经记录了是哪一个 goroutine 持有这个锁。

##### 方案一

获取goroutine id的第一种方式：通过 `runtime.Stack` 方法获取栈帧信息，栈帧信息里包含 goroutine id。

```go

func GetGoroutineID() int {
	var buf [64]byte
	n := runtime.Stack(buf[:], false)
	idField := strings.Fields(strings.TrimPrefix(string(buf[:n]), "goroutine "))[0]
	id, err := strconv.Atoi(idField)
	if err != nil {
		panic(fmt.Sprintf("can't get goroutine id: %v", err))
	}
	return id
}
```

获取goroutine id的第二种方式：获取运行时的 g 指针，反解出对应的 g 的结构。每个运行的 goroutine 结构的 g 指针保存在当前 goroutine 的一个叫做 TLS 对象中。

- 第一步：我们先获取到 TLS 对象；
- 第二步：再从 TLS 中获取 goroutine 结构的 g 指针；
- 第三步：再从 g 指针中取出 goroutine id。

> 需要注意的是，不同 Go 版本的 goroutine 的结构可能不同，所以需要根据 Go 的[不同版本](https://github.com/golang/go/blob/89f687d6dbc11613f715d1644b4983905293dd33/src/runtime/runtime2.go#L412)进行调整。当然了，如果想要搞清楚各个版本的 goroutine 结构差异，所涉及的内容又过于底层而且复杂，学习成本太高。没有必要重复发明轮子，直接使用第三方的库（[petermattis/goid](https://github.com/petermattis/goid)）来获取 goroutine id 就可以了。

手动实现一个可重入的锁：

```go
// RecursiveMutex 包装一个Mutex,实现可重入
type RecursiveMutex struct {
    sync.Mutex
    owner     int64 // 当前持有锁的goroutine id
    recursion int32 // 这个goroutine 重入的次数
}

func (m *RecursiveMutex) Lock() {
    gid := goid.Get()
    // 如果当前持有锁的goroutine就是这次调用的goroutine,说明是重入
    if atomic.LoadInt64(&m.owner) == gid {
        m.recursion++
        return
    }
    m.Mutex.Lock()
    // 获得锁的goroutine第一次调用，记录下它的goroutine id,调用次数加1
    atomic.StoreInt64(&m.owner, gid)
    m.recursion = 1
}

func (m *RecursiveMutex) Unlock() {
    gid := goid.Get()
    // 非持有锁的goroutine尝试释放锁，错误的使用
    if atomic.LoadInt64(&m.owner) != gid {
        panic(fmt.Sprintf("wrong the owner(%d): %d!", m.owner, gid))
    }
    // 调用次数减1
    m.recursion--
    if m.recursion != 0 { // 如果这个goroutine还没有完全释放，则直接返回
        return
    }
    // 此goroutine最后一次调用，需要释放锁
    atomic.StoreInt64(&m.owner, -1)
    m.Mutex.Unlock()
}
```

相当于给 Mutex 打一个补丁，解决了记录锁的持有者的问题，用 owner 字段，记录当前锁的拥有者 goroutine 的 id；recursion 是辅助字段，用于记录重入的次数。

**尽管拥有者可以多次调用 Lock，但是也必须调用相同次数的 Unlock，这样才能把锁释放掉。这是一个合理的设计，可以保证 Lock 和 Unlock 一一对应。**

##### 方案二

方案一是用 goroutine id 做 goroutine 的标识，也可以让 goroutine 自己来提供标识。Go 开发者不期望利用 goroutine id 做一些不确定的东西，所以，没有暴露获取 goroutine id 的方法。

调用者自己提供一个 token，获取锁的时候把这个 token 传入，释放锁的时候也需要把这个 token 传入。通过用户传入的 token 替换方案一中 goroutine id，其它逻辑和方案一一致。

```go
// Token方式的递归锁
type TokenRecursiveMutex struct {
    sync.Mutex
    token     int64
    recursion int32
}

// 请求锁，需要传入token
func (m *TokenRecursiveMutex) Lock(token int64) {
    if atomic.LoadInt64(&m.token) == token { //如果传入的token和持有锁的token一致，说明是递归调用
        m.recursion++
        return
    }
    m.Mutex.Lock() // 传入的token不一致，说明不是递归调用
    // 抢到锁之后记录这个token
    atomic.StoreInt64(&m.token, token)
    m.recursion = 1
}

// 释放锁
func (m *TokenRecursiveMutex) Unlock(token int64) {
    if atomic.LoadInt64(&m.token) != token { // 释放其它token持有的锁
        panic(fmt.Sprintf("wrong the owner(%d): %d!", m.token, token))
    }
    m.recursion-- // 当前持有这个锁的token释放锁
    if m.recursion != 0 { // 还没有回退到最初的递归调用
        return
    }
    atomic.StoreInt64(&m.token, 0) // 没有递归调用了，释放锁
    m.Mutex.Unlock()
}
```

#### 死锁

> 死锁：两个或两个以上的进程（或线程，goroutine）在执行过程中，因争夺共享资源而处于一种互相等待的状态，如果没有外部干涉，它们都将无法推进下去，此时，称系统处于死锁状态或系统产生了死锁。一个经典的死锁问题就是[哲学家就餐问题](https://zh.wikipedia.org/wiki/%E5%93%B2%E5%AD%A6%E5%AE%B6%E5%B0%B1%E9%A4%90%E9%97%AE%E9%A2%98)。

如果想避免死锁，只要破坏这四个条件中的一个或者几个，就可以了。

- 互斥：至少一个资源是被排他性独享的，其他线程必须处于等待状态，直到资源被释放。
- 持有等待：goroutine 持有一个资源，并且还在请求其它 goroutine 持有的资源。
- 不可剥夺：资源只能由持有它的 goroutine 来释放。
- 环路等待：存在一组等待进程，P={P1，P2，…，PN}，P1 等待 P2 持有的资源，P2 等待 P3 持有的资源，依此类推，最后是 PN 等待 P1 持有的资源，这就形成了一个环路等待的死结。

可以引入第三方的锁或者解决持有等待这个问题来避免死锁。
