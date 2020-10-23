# RWMutex

使用 Mutex 来保证读写共享资源的安全性。不管是读还是写，都通过 Mutex 来保证只有一个 goroutine 访问共享资源，这在某些情况下有点“浪费”。在**写少读多**的情况下，即使一段时间内没有写操作，大量并发的读访问也不得不在 Mutex 的保护下变成了串行访问，这个时候，使用 Mutex，对性能的影响就比较大。

区分读写操作：

- 当**读操作**的 goroutine 持有锁，在这种情况下，其它读操作的 goroutine 就不必一直等待，可以并发地访问共享变量，这样可以将串行的读变成并行读，提高读操作的性能。
- 当**写操作**的 goroutine 持有锁，它就是一个排他锁，其它的写操作和读操作的 goroutine，需要阻塞等待持有这个锁的 goroutine 释放锁。

> 这一类并发读写问题叫作 readers-writers 问题，意思就是，同时可能有多个读或者多个写，但是只要有一个线程在执行写操作，其它的线程都不能执行读写操作。

**Go 标准库中的 RWMutex（读写锁）就是用来解决这类 readers-writers 问题的。**

标准库中的 RWMutex 是一个 reader/writer 互斥锁。RWMutex 在某一时刻只能由**任意数量**的 reader 持有，或者是只被**单个**的 writer 持有。

RWMutex 的方法:

- **Lock/Unlock：写操作时调用的方法**。如果锁已经被 reader 或者 writer 持有，那么，Lock 方法会一直阻塞，直到能获取到锁；Unlock 则是配对的释放锁的方法。
- **RLock/RUnlock：读操作时调用的方法**。如果锁已经被 writer 持有的话，RLock 方法会一直阻塞，直到能获取到锁，否则就直接返回；而 RUnlock 是 reader 释放锁的方法。
- **RLocker：这个方法的作用是为读操作返回一个 Locker 接口的对象**。它的 Lock 方法会调用 RWMutex 的 RLock 方法，它的 Unlock 方法会调用 RWMutex 的 RUnlock 方法。

> RWMutex 的零值是未加锁的状态，所以，使用 RWMutex 的时候，无论是**声明变量**，还是**嵌入**到其它 struct 中，**都不必显式地初始化**。

使用示例如下：

```go
// Counter 线程安全的计数器
type Counter struct {
	mu    sync.RWMutex
	count uint64
}

// Incr 计数操作，使用写锁
func (c *Counter) Incr() {
	c.mu.Lock()
	c.count++
	c.mu.Unlock()
}

// Count 返回计数器中的值，使用读锁
func (c *Counter) Count() uint64 {
	c.mu.RLock()
	defer c.mu.RUnlock()
	return c.count
}

func main() {
	var counter Counter
	for i := 0; i < 10; i++ {
		go func() {
            // 每毫秒执行一次查询，通过读写锁，极大提升计数器的性能，
            // 在读取的时候，可以并发进行
			fmt.Println(counter.Count())
			time.Sleep(time.Millisecond)
		}()
	}

	for {
        // 每秒才调用一次，writer 竞争锁的频次比较低
		counter.Incr()
        time.Sleep(time.Second)
	}
}
```

- Incr 方法会修改计数器的值，是一个**写操作**，使用 Lock/Unlock 进行保护。
- Count 方法会读取当前计数器的值，是一个**读操作**，使用 RLock/RUnlock 方法进行保护。

如果使用 Mutex，性能就不会像读写锁这么好。因为多个 reader 并发读的时候，使用互斥锁导致了 reader 要排队读的情况，没有 RWMutex 并发读的性能好。

**当遇到可以明确区分 reader 和 writer goroutine 的场景，且有大量的并发读、少量的并发写，并且有强烈的性能需求，考虑使用读写锁 RWMutex 替换 Mutex。**

> 在实际使用 RWMutex 的时候，如果在 struct 中使用 RWMutex 保护某个字段，一般会把它和这个字段**放在一起**，用来指示两个字段是一组字段。除此之外，还可以采用**匿名字段**的方式嵌入 struct，这样，在使用这个 struct 时，可以直接调用 Lock/Unlock、RLock/RUnlock 方法。

## 实现原理

> RWMutex 是很常见的并发原语，很多编程语言的库都提供了类似的并发类型。RWMutex 一般都是基于**互斥锁**、**条件变量**（condition variables）或者**信号量**（semaphores）等并发原语来实现。

readers-writers 问题基于对读和写操作的优先级，设计和实现分成三类：

- **Read-preferring**：读优先的设计可以提供很高的并发性，但是，在竞争激烈的情况下可能会导致写饥饿。这是因为，如果有大量的读，这种设计会导致**只有所有的读都释放了锁之后，写才可能获取到锁**。
- **Write-preferring**：写优先的设计意味着，如果已经有一个 writer 在等待请求锁的话，它会阻止新来的请求锁的 reader 获取到锁，所以优先保障 writer。当然，如果有一些 reader 已经请求了锁的话，新请求的 writer 也会等待已经存在的 reader 都释放锁之后才能获取。所以，**写优先级设计中的优先权是针对新来的请求而言的**。这种设计主要避免了 writer 的饥饿问题。
- **不指定优先级**：这种设计比较简单，不区分 reader 和 writer 优先级，某些场景下这种不指定优先级的设计反而更有效，因为第一类优先级会导致写饥饿，第二类优先级可能会导致读饥饿，这种不指定优先级的访问不再区分读写，大家都是同一个优先级，解决了饥饿的问题。（这和排他锁好像没啥区别）

Go 标准库中的 RWMutex 是基于 Mutex 实现的，采用的是Write-preferring方案。**一个正在阻塞的 Lock 调用会排除新的 reader 请求到锁。**

```go
type RWMutex struct {
    w           Mutex   // 互斥锁解决多个writer的竞争
    writerSem   uint32  // writer信号量
    readerSem   uint32  // reader信号量
    readerCount int32   // 【双重含义字段】reader的数量（以及是否有 writer 竞争锁）
    readerWait  int32   // writer等待完成的reader的数量
}

const rwmutexMaxReaders = 1 << 30   // 定义最大的 reader 数量
```

### RLock/RUnlock 的实现

```go
func (rw *RWMutex) RLock() {
    if atomic.AddInt32(&rw.readerCount, 1) < 0 {
            // rw.readerCount是负值的时候，意味着此时有writer等待请求锁，
            // 因为writer优先级高，所以把后来的reader阻塞休眠
        runtime_SemacquireMutex(&rw.readerSem, false, 0)
    }
}
func (rw *RWMutex) RUnlock() {
    if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
        rw.rUnlockSlow(r) // 有等待的writer
    }
}
func (rw *RWMutex) rUnlockSlow(r int32) {
    if atomic.AddInt32(&rw.readerWait, -1) == 0 {
        // 最后一个reader了，writer终于有机会获得锁了
        runtime_Semrelease(&rw.writerSem, false, 1)
    }
}
```

readerCount 这个字段有双重含义：

- 没有 writer 竞争或持有锁时，readerCount 和正常理解的 reader 的计数是一样的；
- 有 writer 竞争或持有锁时，readerCount 不仅仅承担着 reader 的计数功能，还能够标识当前是否有 writer 竞争或持有锁:
  - 在调用RLock时，请求锁的 reader 的调用`runtime_SemacquireMutex()`，阻塞等待锁的释放。
  - 在调用RUnlock时，如果它是负值，就表示当前有 writer 竞争锁，调用`rUnlockSlow()`，检查是不是 reader 都释放读锁了，如果读锁都释放了，那么可以唤醒请求写锁的 writer 了

当一个或者多个 reader 持有锁的时候，竞争锁的 writer 会等待这些 reader 释放完，才可能持有这把锁。当 writer 请求锁的时候，是无法改变既有的 reader 持有锁的现实的，也不会强制这些 reader 释放锁，它的优先权只是限定后来的 reader 不要和它抢。所以，rUnlockSlow 将持有锁的 reader 计数减少 1 的时候，会检查既有的 reader 是不是都已经释放了锁，如果都释放了锁，就会唤醒 writer，让 writer 持有锁。

### Lock

RWMutex 是一个多 writer 多 reader 的读写锁，所以同时可能有多个 writer 和 reader。那么，为了避免 writer 之间的竞争，RWMutex 就会使用一个 Mutex 来保证 writer 的互斥。

```go
func (rw *RWMutex) Lock() {
    // 首先解决其他writer竞争问题
    rw.w.Lock()
    // 反转readerCount，告诉reader有writer竞争锁
    r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
    // 如果当前有reader持有锁，那么需要等待
    if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
        runtime_SemacquireMutex(&rw.writerSem, false, 0)
    }
}
```

一旦一个 writer 获得了内部的互斥锁，就会反转 readerCount 字段，把它从原来的正整数 `readerCount(>=0)` 修改为负数（`readerCount-rwmutexMaxReaders`），让这个字段保持两个含义（既保存了 reader 的数量，又表示当前有 writer），还会记录当前活跃的 reader 数量，所谓活跃的 reader，就是指持有读锁还没有释放的那些 reader。

如果 readerCount 不是 0，就说明当前有持有读锁的 reader，RWMutex 需要把这个当前 readerCount 赋值给 readerWait 字段保存下来， 同时，这个 writer 进入阻塞等待状态。每当一个 reader 释放读锁的时候（调用 RUnlock 方法时），readerWait 字段就减 1，直到所有的活跃的 reader 都释放了读锁，才会唤醒这个 writer。

### Unlock

```go
func (rw *RWMutex) Unlock() {
    // 告诉reader没有活跃的writer了
    r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)

    // 唤醒阻塞的reader们
    for i := 0; i < int(r); i++ {
        runtime_Semrelease(&rw.readerSem, false, 0)
    }
    // 释放内部的互斥锁
    rw.w.Unlock()
}
```

当一个 writer 释放锁的时候，它会再次反转 readerCount 字段。

> 因为当前锁由 writer 持有，所以，readerCount 字段是反转过的，并且减去了 rwmutexMaxReaders 这个常数，变成了负数。所以，这里的反转方法就是给它增加 rwmutexMaxReaders 这个常数值。

既然 writer 要释放锁了，那么就需要唤醒之后新来的 reader，不必再阻塞它们了，让它们继续执行。在 RWMutex 的 Unlock 返回之前，需要把内部的互斥锁释放。释放完毕后，其他的 writer 才可以继续竞争这把锁。

注意点：

1. 理解 readerCount 这个字段的含义以及反转方式。
2. 注意字段的更改和内部互斥锁的顺序关系。
   1. 在 Lock 方法中，是先获取内部互斥锁，才会修改的其他字段；
   2. 而在 Unlock 方法中，是先修改的其他字段，才会释放内部互斥锁，这样才能保证字段的修改也受到互斥锁的保护。

## 踩坑点

### 不可复制

RWMutex 是由一个互斥锁和四个辅助字段组成的。因为互斥锁是不可复制的，再加上四个有状态的字段，RWMutex 就更加不能复制使用。

不能复制的原因和互斥锁一样。

- 一旦读写锁被使用，它的字段就会记录它当前的一些状态。这个时候复制这把锁，就会把它的状态也给复制过来。
- 但是，原来的锁在释放的时候，并不会修改复制出来的这个读写锁，这就会导致复制出来的读写锁的状态不对，可能永远无法释放锁。

解决方案也和互斥锁一样，借助 vet 工具，在变量赋值、函数传参、函数返回值、遍历数据、struct 初始化等时，检查是否有读写锁隐式复制的情景。

### 重入导致死锁

读写锁因为重入（或递归调用）导致死锁的情况更多：

1. 因为读写锁内部基于互斥锁实现对 writer 的并发访问，而互斥锁本身是有重入问题的，所以，writer 重入调用 Lock 的时候，就会出现死锁的现象。
2. 有活跃 reader 的时候，writer 会等待，如果在 reader 的读操作时调用 writer 的写操作（它会调用 Lock 方法），那么，这个 reader 和 writer 就会形成**互相依赖的死锁状态**。Reader 想等待 writer 完成后再释放锁，而 writer 需要这个 reader 释放锁之后，才能不阻塞地继续执行。这是一个读写锁常见的死锁场景。
3. 当一个 writer 请求锁的时候，如果已经有一些活跃的 reader，它会等待这些活跃的 reader 完成，才有可能获取到锁，但是，如果之后活跃的 reader 再依赖新的 reader 的话，这些新的 reader 就会等待 writer 释放锁之后才能继续执行，这就形成了一个**环形依赖**： writer 依赖活跃的 reader -> 活跃的 reader 依赖新来的 reader -> 新来的 reader 依赖 writer。

第三种死锁的情况相当的隐蔽，原因在于和 RWMutex 的设计和实现有关。来看一个计算阶乘 (n!) 的例子：

### 释放未加锁的RWMutex

和互斥锁一样，Lock 和 Unlock 的调用总是成对出现的，RLock 和 RUnlock 的调用也必须成对出现。

- Lock 和 RLock 多余的调用会导致锁没有被释放，可能会出现**死锁**，
- 而 Unlock 和 RUnlock 多余的调用会导致 **panic**。

在生产环境中出现 panic 是大忌，在使用读写锁的时候，一定要注意，**不遗漏不多余**。
