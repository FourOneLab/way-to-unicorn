---
title: 22-bytes
date: 2019-11-25T11:15:47.526182+08:00
draft: false
---

strings包和bytes包在API方面非常相似，单从它们提供的函数的数量和功能上说，差别微乎其微。

- strings包主要面向的是Unicode字符和经过UTF-8编码的字符串
- bytes包主要面向的是字节和字节切片

## `bytes.Buffer`

```go
// Buffer是一个具有读写方法的可变大小的字节缓冲区。
// 零值的Buffer是一个准备使用的空缓冲区。
type Buffer struct {
    buf     []byte // contents are the bytes buf[off : len(buf)]
    off      int    // read at &buf[off], write at &buf[len(buf)]
    lastRead readOp // last read operation, so that Unread* can work correctly.
}
```

**主要用途，作为字节序列的缓冲区**。与`strings.Builder`一样，`bytes.Buffer`也是开箱即用。

- `strings.Builder`：只能拼接和导出字符串
- `bytes.Buffer`：不但可以拼接、截断其中的字节序列，以各种形式导出其中的内容，还可以顺序读取其中的子序列

> 在内部，`bytes.Buffer`类型同样使用字节切片作为内容容器，并且与`strings.Reader`类型类似，`bytes.Buffer`有一个int类型的字段，用于表示已读字节的计数。**已读字节的计数无法通过`bytes.Buffer`提供的方法计算出来**。

```go
var buffer1 bytes.Buffer
contents := "Simple byte buffer for marshaling data."
fmt.Printf("Writing contents %q ...\n", contents)
buffer1.WriteString(contents)
fmt.Printf("The length of buffer: %d\n", buffer1.Len()) // 返回其中未被读取部分的长度，而不是已存内容的总长度
fmt.Printf("The capacity of buffer: %d\n", buffer1.Cap())

// Output
Writing contents "Simple byte buffer for marshaling data."
The length of buffer: 39    // 空格，标点，字符加起来，未调用任何读取方法，因此已读计数为零
The capacity of buffer: 64  // 根据切片自动扩容策略，64看起来也很合理
```

```go
p1 := make([]byte, 7)
n, _ := buffer1.Read(p1)    // 从buffer中读取内容，并用它们填满长度为7的字节切片
fmt.Printf("%d bytes were read. (call Read)\n", n)
fmt.Printf("The length of buffer: %d\n", buffer1.Len())
fmt.Printf("The capacity of buffer: %d\n", buffer1.Cap())

// Output
7 bytes were read. (call Read)
The length of buffer: 32
The capacity of buffer: 64
```

- Buffer 值的长度：**Buffer值的长度是未读内容的长度，而不是已存在内容的总长度**。它与在当前值之上的读操作和写操作都有关系，随着这两种操作的进行而改变。
- Buffer值的容量：Buffer值的容量是指它的内容容器（字节切片）的容量，它只与在当前值之上的写操作有关，并会随着内容的写入而不断增长。
- 已读字节计数：在`strings.Reader`中有Size方法可以得出内容长度的值，所以用内容长度减去未读部分长度就是已读计数，但是`bytes.Buffer`类型没有这样的方法，它的Cap方法提供的是内容容器的容量，而不是内容的长度，大部分情况下这两者是不同的，因此很难估计Buffer值的已读计数。

### 已读字节计数的作用

`bytes.Buffer`中的已读计数的大致功能如下：

1. 读取内容时，相应方法会根据已读计数找到未读部分，并在读取后更新计数
2. 写入内容时，如需扩容，相应方法会根据已读计数实现扩容策略
3. 截断内容时，相应方法截掉的是已读计数代表索引之后的未读部分
4. 读回退时，相应方法需要用已读计数记录回退点
5. 重置内容时，相应方法会把已读计数重置为0
6. 导出内容时，相应方法只会导出已读计数代表的索引之后的未读部分
7. 获取长度时，相应方法会依据已读计数和内容容器的长度，计算未读部分的长度并返回

**已读计数在`bytes.Buffer`类型中的重要作用，大部分方法都用到它**。

#### 读取内容

1. 相应方法先根据已读计数，判断一下内容容器中是否还有未读的内容
2. 如果有，那么它就会从已读计数代表的索引处开始读取
3. 读取完成后，及时更新已读计数，也就是说会记录一下又有多少字节被读取了

> 读取内容的相应方法：包括所有名称以Read开头的方法，已经Next方法和WriteTo方法。

#### 写入内容

1. 绝大多数的相应方法都会先检查当前的内容容器，是否有足够的容量容纳新的内容
2. 如果没有，就对内容容器进行扩容
3. 扩容的时候，方法会在必要时，依据已读计数找到未读部分，把其中的内容拷贝到扩容后的内容容器的头部位置
4. 然后方法把已读计数的值重置为0，以表示下一次读取需要从内容容器的第一个字节开始

> 写入内容的相应方法：包括所有名称以Write开头的方法，以及ReadFrom方法。

#### 截取内容

截取内容的方法`Truncate`，它接收一个int类型的参数，这个参数的值代表了：在截取时需要保留头部的多少个字节。

> 这里的头部，指的并不是内容容器的头部，而是其中未读部分的头部。头部的起始索引正是由已读计数的值表示的。在这种情况下，已读计数的值再加上参数值后得到的和，就是内容容器新的总长度。

#### 读回退

在`bytes.Buffer`中，用于读回退的方法有：

- UnreadByte：
  - 回退一个字节
  - 实现方法是把已读计数减一
- UnreadRune：
  - 回退一个Unicode字符
  - 实现方法是在已读计数中减去上一个被读取的Unicode字符所占用的字节数
  - 这个字节数有`bytes.Buffer`的另一个字段负责存储，它在这里的有效取值范围是[1,4]，只有ReadRune方法才会把这个字段的值设定在此范围之内
    > 只有ReadRune方法之后，UnreadRune方法的调用才会成功，该方法比UnreadByte方法的适用面更窄。

调用它们一般都是为了退回上一次被读取内容末尾的那个分隔符，或者在重新读取前一个字节或字符做准备。

> 退回的前提是，在调用它们之前的那个操作必须是“读取”，并且是成功的读取，否则这些方法只能忽略后续操作并返回一个非nil的错误值。

`bytes.Buffer`的Len方法返回的是内容容器中未读部分的长度，而不是其中已存内容的总长度，该类型的Bytes方法和String方法的行为与Len方法的行为保存一直，只会访问未读部分中的内容，并返回相应的结果值。

**在已读计数代表的索引之前的那些内容，永远都是已经被读过的，它们几乎没有机会再次被读取**。这些已读内容所在的内存空间可能会被存入新的内容，这一般都是由于重置或者扩充内容容器导致的。这时，已读计数一定会被置为0，从而再次指向内容容器中的第一个字节。这有时候也是为了避免内存分配和重用内存空间。

### 扩容策略

Buffer值既可以手动扩容，也可以自动扩容，这两种扩容方式的策略基本一直，除非完全确定后续内容所需的字节数，否则让Buffer值自动扩容就好了。

扩容时，扩容代码会先判断内容容器的剩余容量，是否可以满足调用方的要求，或者是否足够容纳新的内容。如果可以，扩容代码会在当前的内容容器之上，进行长度扩充。

1. 如果内容容器的容量与其长度的差，大于或等于另需的字节数，那么扩容代码就会通过切片操作对原有的内容容器的长度进行扩充，如下所示：

    ```go
    b.buf = b.buf[:length+need]
    ```

2. 如果内容容器剩余的容量不够了，那么扩容代码可能就会用新的内容容器去代替原有的内容容器，从而实现扩容。此处进行一步优化。
   1. 如果当前内容容器的容量的一半，仍然大于或等于其现有长度再加上另需的字节数的和，即`cap(b.buf)/2 >= len(b.buf)+need`，那么扩容代码就会服用现有的内容容器，并把容器中的未读内容拷贝到它的头部位置。**这样已读的内容会被全部未读的内容和之后的新内容覆盖掉**。这样的复用至少节省一次扩容带来的内存分配以及若干字节的拷贝。
   2. 若不满足优化条件，即当前内容容器的容量小于新长度的两倍。那么扩容代码就只能创建一个新的内容容器，并把原有容器中的未读内容拷贝进去，最后再用新的内容容器替换原来的。
    > 新容器的容量=2×原来容量+所需字节数

通过上述步骤，对内容容器的扩容基本完成，不过为了内部数据的一致性，以及避免原有的已读内容造成的数据混乱，扩容代码会把已读计数重置为0，并再对内容容器做一次切片操作，以掩盖掉原有的已读内容。

> 对于处于零值状态的Buffer值来说，如果第一次扩容时的另需字节数不大于64，那么该值会基于一个预先定义好的，长度为64的字节数组来创建内容容器，这样内容容器的容量就是64，**这样做的目的是为了让Buffer值在刚被真正使用的时候就可以快速地做好准备**。

### 哪些方法可能造成内容泄露

> 内容泄露是指使用Buffer值的一方通过某种非标准方式等到了本不该得到的内容。

比如，通过调用Buffer值的某个用于读取内容的方法，得到了一部分未读内容，我们应该也只应该通过这个方法的结果值，拿到在那一时刻值中的未读内容，但是，在这个Buffer值又有了一些新的内容之后，却可以通过当时得到的结果值，直接获得新的内容，而不需要再次调用相应的方法。

**这就是典型的非标准读取方式**。这种读取方式不应存在，如果存在，不应该使用。

在`bytes.Buffer`中，Bytes方法和Next方法都可能会造成内容的泄露，原因在于，它们都把基于内容容器的切片直接返回给了方法的调用方。

> **基于切片可以直接访问和操作它的底层数组，不论这个切片是基于某个数组得来的，还是通过另一个切片操作获得的**。

在这里Bytes方法和Next方法返回的字节切片，都是通过对内容容器做切片操作得到的，它们与内容容器共用同一个底层数组，起码在一段时间内是这样的。

#### 例子

Bytes方法会返回在调用那一刻它所属值中的所以未读内容：

```go
contents := "ab"
buffer1 := bytes.NewBufferString(contents)
fmt.Printf("The capacity of new buffer with contents %q: %d\n", contents, buffer1.Cap())
// 内容容器的容量为：8。
unreadBytes := buffer1.Bytes()
fmt.Printf("The unread bytes of the buffer: %v\n", unreadBytes) // 未读内容为：[97 98]。


buffer1.WriteString("cdefg")// 在向buffer1中写入字符串值“cdefg”
fmt.Printf("The capacity of buffer: %d\n", buffer1.Cap()) // 内容容器的容量仍为：8。
// 通过简单的在切片操作，就可以利用这个结果值拿到buffer1在此时的所有未读内容
unreadBytes = unreadBytes[:cap(unreadBytes)]
fmt.Printf("The unread bytes of the buffer: %v\n", unreadBytes) // 基于前面获取到的结果值可得，未读内容为：[97 98 99 100 101 102 103 0]。

// 如果将unreadBytes的值传到外界，那么就可以通过该值操纵buffer1的内容
unreadBytes[len(unreadBytes)-2] = byte('X') // 'X'的 ASCII 编码为 88。
fmt.Printf("The unread bytes of the buffer: %v\n", buffer1.Bytes()) // 未读内容变为了：[97 98 99 100 101 102 88]。
```

Next方法也有这样的问题。

**如果经过扩容，Buffer值的内容容器或者它的底层数组被重新设定了，那么之前的内容泄露问题就无法再进一步发展了**。在传出切片这类值之前要做好隔离，比如，先对它们进行深度拷贝，然后再把副本传出来。
