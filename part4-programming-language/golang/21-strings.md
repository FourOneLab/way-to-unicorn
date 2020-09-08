---
title: 21-strings
date: 2019-11-25T11:15:47.526182+08:00
draft: false
---

Golang不但拥有可以独立代表Unicode字符的类型rune，而且还可以对字符串值进行Unicode字符拆分的for语句。标准库Unicode包及其子包还提供了很多函数和数据类型，可以解析各种内容中的Unicode字符。

标准库中的strings代码包，用到了unicode包和`unicode/utf8`包中的程序，如：

- `strings.Builder`类型的WriteRune方法
- `strings.Reader`类型的ReadRune方法

## 与string值相比`strings.Builder`类型的优势

1. 已存在的内容不可变，但可以拼接更多的内容
2. 减少内存分配和内容拷贝的次数
3. 可将内容充值，可重用值

### string类型

在Golang中，string类型的值是不可变的，如果想要获得一个不一样的字符串，只能基于原来的字符串进行裁剪、拼接等操作，从而生成一个新的字符串。

- 裁剪：可以使用切片表达式
- 拼接：可以使用操作符“+”实现

在底层，一个string值的内容会被存储到一块连续的内存空间中，同时，这块内存容纳的字节数量会被记录下来，并用于表示该string值的长度。**可以把这块内存的内容看成一个字节数组，而相应的string值则包含了指向字节数组头部的指针值**。这样在string值上应用切片表达式，就相当于在对其底层的字节数组做切片。

在进行字符串拼接的时候，Golang会把所有被拼接的字符串依次拷贝到一个崭新且足够大的连续内存空间中，并把持有相应指针的string值作为结果返回。**当程序中存在过多的字符串拼接操作的时候，会对内存分配产生非常大的压力**。

> 虽然string值在内部持有一个指针，但其类型仍然属于值类型，由于string值的不可变，其中的指针值也为内存空间的节省做出了贡献。

**一个string值会在底层与它所有的副本共用同一个字节数组，由于这里的字节数组永远不会被改变，所有这样是绝对安全的**。

### `strings.Builder`类型

与string相比`strings.Builder`值的优势体现在字符串拼接方面。Builder值中有一个用于承载内容的容器（简称内容容器），它是一个以byte为元素类型的切片（字节切片），由于这样的字节切片的底层数组就是一个字节数组，它与string值存储内容的方式是一样的。

实际上它们都是通过unsafe.Pointer类型的字段来持有那个指向了底层字节数组的指针值。因为这样的构造Builder值拥有高校利用内存的前提条件。

> 对于字节切片本身来说，它包含的任何元素值都可以被修改，但是Builder值并不允许这样做，其中的内容只能够被拼接或者被重置。这就意味着Builder值中的内容是不可变的。因此，利用Builder值提供的方法（Write、WriteByte、WriteRune、WriteString）拼接更多的内容，而丝毫不用担心这些方法会影响到已存在的内容。

通过调用上述方法把新的内容拼接到已存在的内容的尾部，如果有必要，Builder会自动地对自身的内容容器进行扩容，**这里的自动扩容策略与切片的扩容策略一致**。内容容器的容量足够时不会进行扩容，没有扩容，那么已存在的内容就不会被拷贝。

通过Builder的Grow方法可以进行手动扩容，它接收一个int类型的参数n，该参数用于代表将要扩充的字节数量。Grow方法会把其所属的内容容器的容量增加n个字节，它会生成一个字节切片作为新的内容容器，该切片的容量会是原容器容量的两被在加上n，之后会把原容器中的所有字节全部拷贝到新容器中。

```go
var builder1 strings.Builder
// 省略若干代码。
fmt.Println("Grow the builder ...")
builder1.Grow(10)
fmt.Printf("The length of contents in the builder is %d.\n", builder1.Len())
// 如果当前内容容器的未用容量已经够用，即未用容量 >=n ,那么Grow方法什么也不做

fmt.Println("Reset the builder ...")
// 调用Reset方法，可以让Builder值重新回到零值状态，就像它从未被使用过那样
// 一旦被重置，Builder值中原有的内容容器会被直接丢弃
// 与其他的所有内容，将会被Go语言的垃圾回器标记并回收掉
builder1.Reset()
fmt.Printf("The third output(%d):\n%q\n", builder1.Len(), builder1.String())
```

## `strings.Builder`类型的使用约束

- 在已被真正使用后就不可在被复制
    > 调用Builder值的拼接方法和扩容方法就意味着开始真正使用它了，因为这些方法都会改变其所属值中的内容容器的状态，一旦调用了它们，就不能再以任何的方式对其所属值进行复制了，否则只要在任何副本上调用上述方法都会引发panic。

    > 这种panic会告诉我们，这样的使用方式并不合法，因为这里的Builder值是副本而不是原值，这的复制方法包括但不限于函数间传递值、通过通道传递值、把值赋予变量等

    ```go
    var builder1 strings.Builder
    builder1.Grow(1)
    builder3 := builder1
    //builder3.Grow(1) // 这里会引发 panic。
    _ = builder3
    ```

    由于Builder值不能在被复制，所以肯定不会出现多个Builder值中的内容容器公用一个底层字节数组的情况，这样避免了多个同源的Builder值在拼接内容时可能产生的冲突问题。

    虽然已经使用的Builder值不能再被复制，但是它的指针值却可以，无论什么时候，都可以通过任何方式复制这样的指针值。这样的指针值指向的都会是同一个Builder值。

    ```go
    f2 := func(bp *strings.Builder) {
    (*bp).Grow(1) // 这里虽然不会引发 panic，但不是并发安全的。
    builder4 := *bp
    //builder4.Grow(1) // 这里会引发 panic。
    _ = builder4
    }
    f2(&builder1)
    ```

    这就产生了一个问题，如果Builder值被多方同时操作，那么其中的内容就可能产生混乱，这就是所说的操作冲突和并发安全问题。

- 由于其内容不是完全不可变，所以需要使用方自行解决操作冲突和并发安全问题

    1. Builder值自己是无法解决这些问题的，所以在通过传递其指针值共享Builder值的时候，一定要确保各方对它的使用是正确的、有序的，并且是并发安全的，**彻底的解决方案是，绝对不要共享Builder值以及它的指针值**。
    2. 可以在各处分别声明一个Builder值来使用，也可以先声明一个Builder值，然后在真正使用它之前，便将它的副本传到各处。
    3. 先使用在传递，只要在传递之前调用它的Reset方法即可。

    ```go
    builder1.Reset()
    builder5 := builder1
    builder5.Grow(1) // 这里不会引发 panic。
    ```

关于复制Builder值的约束是有意义的，也是很有必要的。虽然仍然在通过某些方式共享Builder值，但最好还是不要以身犯险，各自为政是最好的解决方案。**对于处在零值状态的Builder值，复制不会有任何问题**。

## `strings.Reader`类型的值如何高效读取字符串

与`strings.Builder`类型恰恰相反，`strings.Reader`类型是为了高效读取字符串而存在的。它的高效体现在**它对字符串的读取机制上，它封装了很多用于在string值上读取内容的最佳实践**。

`strings.Reader`类型的值（简称Reader值），可以让我们很方便地读取一个字符串中的内容，在读取的过程中，Reader值会保存已读取的字节的计数（简称已读计数）。

- 已读计数，代表这下一个读取的起始索引为止，Reader值就是依靠这样的一个计数，以及针对字符串的切片表达式，从而实现快读速读取的。

- 已读计数，也是读取回退和位置设定时的重要依据。虽然它属于Reader的内部结构，但我们还是可以通过该值的Len方法和Size方法把它计算出来的。如下代码所示：

```go
var reader1 strings.Reader
// 省略若干代码。
readingIndex := reader1.Size() - int64(reader1.Len()) // 计算出的已读计数。
```

Reader值拥有的大部分用于读取的方法都会及时更新已读计数，比如：

- ReadByte方法会在读取成功后将这个计数的值加1
- ReadRune方法读取成功之后，会把被读取的字符所占用的字节数作为计数的增量
- Seek方法也会更新该值的已读计数器，它的主要擢用正是设定下一次读取的起始索引位置。如果把`io.SeekCurrent`的值作为第二个参数传给该方法，那么它会依据当前的已读计数，以及第一个参数offset的值来计算新的计数值。Seek方法会返回新的计数值，所以我们可以很容易地验证这一点，如下所示。

    ```go
    offset2 := int64(17)
    expectedIndex := reader1.Size() - int64(reader1.Len()) + offset2
    fmt.Printf("Seek with offset %d and whence %d ...\n", offset2, io.SeekCurrent)
    readingIndex, _ := reader1.Seek(offset2, io.SeekCurrent)
    fmt.Printf("The reading index in reader: %d (returned by Seek)\n", readingIndex)
    fmt.Printf("The reading index in reader: %d (computed by me)\n", expectedIndex)
    ```

> ReadAt方法是一个例外，它既不会依据已读计数进行读取，也不会在读取之后更新它。因此这个方法可以自由地读取其所属的Reader值中的任何内容。

综上所属，Reader值实现高效读取的关键在于它内部的已读计数，计数的值就代表这下一次读取的起始索引位置，它可以很容地被计算出来，Reader值的Seek方法可以直接设定该值中的已读计数值。
