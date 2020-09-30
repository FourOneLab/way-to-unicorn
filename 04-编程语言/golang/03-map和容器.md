# Map和容器

## Map

```go
type Map struct {
	key, elem Type
}

// Type表示Go的类型
// 所有类型都实现Type接口。
type Type interface {
	// Underlying 返回类型的基础类型
	Underlying() Type

	// String 返回类型的字符串表示形式
	String() string
}
```

map中存储的不是单一值的集合，而是键值对的集合。

> 一个键和一个值，分别代表了一个从属于某一类型的独立值，把它们两个捆绑在一起就是一个键值对。

### 键的类型约束

Go语言的map其实是一个哈希表的特定实现，在这个实现中，**键的类型是受限的（不支持函数类型、map类型和切片类型）**，**值可以是任意类型**。把键理解为元素的一个索引，在哈希表中通过键查找与它成对的那个元素。键与值的对应关系称为映射，哈希表的映射过程就存在于对键值对的增删改查操作中。

> 映射过程的第一步就是键值转换为哈希值：map中的每个键都由它的哈希值代表，map不会独立存储任何键值，但会独立存储它的哈希值。

Go语言规范规定，在键类型的值之间，必须可以使用操作符`==`和`！=`，也就是必须支持判等操作。

- 如果键的类型是接口类型，那么键值的实际类型也不能是上述三种类型，否则程序运行中会引发panic。
- 如果键的类型是数组类型，那么确保该类型的元素类型不是上述三种类型。
- 如果键的类型是结构体，那么保证其中字段类型的合法性。

因为”哈希碰撞“的存在，Go语言首先用键的哈希值去进行比对，如果键的哈希值相同，在用键本身去比对，如果键类型的值之间无法判等，那么这个映射过程就无法继续进行。

### 键类型的优先选择

在映射的过程中，有两个重要且耗时的操作：

1. 把键值转换为哈希值
2. 把要查找的键值与哈希桶中的键值做对比

但从性能的角度，求哈希和判等操作的速度越快，对应的类型就越适合作为键类型。Go语言中所有的基本类型、指针类型、数组类型、结构体类型和接口类型都有一套各自的算法，其中包含哈希和判等。

基本类型的哈希算法，类型的宽度（单个值需要占用的字节数）越小的类型求哈希的速度越快，优先选择数值类型和指针类型，通常情况下类型的宽度越小越好。

高级类型的哈希算法：

1. 对**数组**类型值的求哈希实际上是依次求得它的每个元素的哈希值并进行合并，所以速度取决于它的元素类型以及它的长度。
2. 对于**结构体**类型的值求哈希实际上就是它的所有字段求哈希并进行合并，关键在于各个字段的类型以及字段的数量。
3. 对于**接口**类型，具体的哈希算法由值的实际类型决定。

不建议使用高级类型作为map的键，不仅因为求哈希和判等速度慢，而且它们中的值存在变数。除了添加键值对，在一个值为nil的map上做任何操作都不会引起错误。

## container

container包中的容器都**不是线程安全**的。

### heap

heap 是一个堆的实现。一个堆正常保证了获取/弹出最大（最小）元素的时间为`o(logn)`、插入元素的时间为`o(logn)`。

堆实现接口如下：

```golang
// src/container/heap.go
type Interface interface {
    sort.Interface
    Push(x interface{}) // add x as element Len()
    Pop() interface{} // remove and return element Len() - 1.
}
```

heap 是基于 `sort.Interface` 实现的:

```golang
// src/sort/
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}
```

因此，如果要使用官方提供的 heap，需要实现如下几个接口：

```go
Len() int {} // 获取元素个数
Less(i, j int) bool  {} // 比较方法
Swap(i, j int) // 元素交换方法
Push(x interface{}){} // 在末尾追加元素
Pop() interface{} // 返回末尾元素
```

然后可以使用如下几种方法：

```go
// 初始化一个堆
func Init(h Interface){}
// push一个元素倒堆中
func Push(h Interface, x interface{}){}
// pop 堆顶元素
func Pop(h Interface) interface{} {}
// 删除堆中某个元素，时间复杂度 log n
func Remove(h Interface, i int) interface{} {}
// 调整i位置的元素位置（位置i的数据变更后）
func Fix(h Interface, i int){}
```

使用`container/heap`实现堆和优先队列，这里实现的堆的底层都是用数组来保存数据：

```go
package main

import (
	"container/heap"
	"fmt"
)

// 数据类型是int的小顶堆
type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

// Push和Pop操作使用指针接收器，因为这两个操作修改了切片的长度而不仅仅是内容
func (h *IntHeap) Push(x interface{}) {	*h = append(*h, x.(int))}

func (h *IntHeap) Pop() interface{} {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[0 : n-1]
	return x
}

func main() {
	h := &IntHeap{2, 1, 5}  // 声明并初始化一个切片指针
	heap.Init(h)    // 先堆化
	heap.Push(h, 3) // 插入数据
	fmt.Printf("minimum: %d\n", (*h)[0])
	for h.Len() > 0 {
		fmt.Printf("%d ", heap.Pop(h))  // 获取堆顶元素
	}
}
```

```go
package main

import (
	"container/heap"
	"fmt"
)

// Item是优先队列中的一个元素
type Item struct {
	value    string // item中保存的值
	priority int    // item在队列中的优先级
	// index主要用于修改item的优先级之后的更新操作
	index int // item在队列中的位置
}

// 一个报错Item类型数据的优先队列
type PriorityQueue []*Item

func (pq PriorityQueue) Len() int { return len(pq) }

// 优先队列的堆顶元素是优先级最高的（数值最大），因此是一个大顶堆
func (pq PriorityQueue) Less(i, j int) bool { return pq[i].priority > pq[j].priority }

func (pq PriorityQueue) Swap(i, j int) {
	pq[i], pq[j] = pq[j], pq[i]
	pq[i].index = i
	pq[j].index = j
}

func (pq *PriorityQueue) Push(x interface{}) {
	n := len(*pq)
	item := x.(*Item)
	item.index = n
	*pq = append(*pq, item)
}

func (pq *PriorityQueue) Pop() interface{} {
	old := *pq
	n := len(old)
	item := old[n-1]
	old[n-1] = nil  // 防止内存泄露
	item.index = -1 // 安全起见，因为堆底层数组的索引从0开始
	*pq = old[0 : n-1]
	return item
}

// update修改优先队列中item的值和优先级
func (pq *PriorityQueue) update(item *Item, value string, priority int) {
	item.value = value
	item.priority = priority
	heap.Fix(pq, item.index)
}

func main() {
	// 初始化几个item元素
	items := map[string]int{
		"banana": 3, "apple": 2, "pear": 4,
	}


    // 创建一个优先队列并将item依次放入数组中，并调用Init进行堆化
	pq := make(PriorityQueue, len(items))
	i := 0
	for value, priority := range items {
		pq[i] = &Item{
			value:    value,
			priority: priority,
			index:    i,
		}
		i++
	}
	heap.Init(&pq)

// 往优先队列中添加一个新的item元素
	item := &Item{
		value:    "orange",
		priority: 1,
	}
    heap.Push(&pq, item)
    // 更新这个item元素的优先级和值
	pq.update(item, item.value, 5)

	// 按照优先级顺序从优先队列中输出
	for pq.Len() > 0 {
		item := heap.Pop(&pq).(*Item)
		fmt.Printf("%.2d:%s ", item.priority, item.value)
	}
}

```

### list

Go语言的链表实现在标准库的`container/list`代码包中。代码包中有两个公开的程序实体：List（双向链表）和Element（链表中的元素）。

```go
// List结构体
type List struct {
	root Element // 哨兵节点，只使用&root, root.prev 和 root.next
	len  int     // 当前列表长度，不包括哨兵节点

// Element结构体
type Element struct {
    // 双链表中的前驱和后继指针，为了简化实现，list被实现为一个循环双链表
    // 所以，&l.root即使最后一个元素（l.Back()）的下一个节点，
    // 也是第一个元素（l.Front()）的前一个节点
	next, prev *Element

	// 表示元素属于哪一个链表
	list *List

	// 元素中存储的值
	Value interface{}
}
```

```go
// 把给定元素移动到另一个元素的前面或后面
func (l *List) MoveBefore(e, mark *Element)
func (l *List) MoveAfter(e, mark *Element)

// 把给定元素移动到链表的最前端和最后端
func (l *List) MoveToFront(e *Element)
func (l *List) MoveToBack(e *Element)
```

在List包含的方法中，用于插入新元素的那些方法都只接受`Interface{}`类型的值。这些方法在内部会使用Element值，包装接收到的新元素。**这样子为了避免链表的内部关联遭到外界破坏**。

```go
func (l *List) Front() *Element     //返回链表最前端元素的指针
func (l *List) Back() *Element      //返回链表最后段元素的指针

func (l *List) InsertBefore(v interface{}, mark *Element) *Element      //在链表指定元素前插入元素并返回插入元素的指针
func (l *List) InsertAfter(v interface{}, mark *Element) *Element       //在链表指定元素后插入元素并返回插入元素的指针

func (l *List) PushFront(v interface{}) *Element        //将指定元素插入到链表头
func (l *List) PushBack(v interface{}) *Element         //将指定元素插入到链表尾
```

List和Element都是结构体类型，**结构体类型的特点是它们的零值都拥有特定结构，但是没有任何定制化内容的值，值中的字段都被赋予各自类型的零值**。

> 只做声明却没有初始化的变量被赋予该类型的零值，每个类型的零值都会依据该类型的特性而被设定。

```go
var l list.List
// 声明的变量l是一个长度为0的双链表
```

这样的链表可以开箱即用的原因在于“延迟初始化”机制。

延迟初始化的优点：

把初始化操作延后，仅在实际需要的时候才进行，延迟初始化的优点在于“延后”，**它可以分散初始化操作带来的计算量和存储空间消耗**。

> 如果需要集中声明非常多的大容量切片，那么CPU和内存的使用量肯定会激增，并且只要设法让其中的切片和底层数组被回收，内存使用量才会有所下降。如果数组可以被延迟初始化，那么CPU和内存的压力就被分散到实际使用它们的时候，这些数组被实际使用的时间越分散，延迟初始化的优势就越明显。

延迟初始化的缺点：

延迟初始化的缺点也在于“延后”，如果在调用链表的每个方法的时候，都需要先去判断链表是否已经被初始化，这也是计算量上的浪费，这些方法被非常频繁地调用的情况下，这种浪费的影响就开始明显，程序的性能会降低。

解决方案：

1. 在链表的实现中，一些方法无需对是否初始化做判断，如`Front()`和`Back()`方法，一旦发现链表的长度为0，直接返回nil
2. 在插入、删除或移动元素的方法中，只要判断传入的元素中指向所属链表的指针，是否与当前链表的指针相等就可以

> `PushFront()`、`PushBack()`、`PushBackList()`、`PushFrontList()`方法总是会**先判断链表的状态**，并在必要时进行延迟初始化。在向新链表中添加新元素时，肯定会调用这四个方法之一。

### ring

`container/ring`包中的Ring类型实现的是一个循环链表（环）。

```go
type Ring struct {
	next, prev *Ring
	Value      interface{}
}
```

> List在内部就是一个循环链表，它的根元素永远不会持有任何实际的元素值，而该元素的存在是为了连接这个循环链表的首尾两端。List的零值是一个只包含哨兵节点，不包含任何实际元素值的空链表，计算长度时不会包含它。

| 差异                 | Ring                                                       | List                                      |
| -------------------- | ---------------------------------------------------------- | ----------------------------------------- |
| 表示方式、结构复杂度 | Ring类型的数据结构仅由它自身即可代表                       | List类型需要它及Element类型联合表示       |
| 表述维度             | Ring类型的值，严格来说只代表起所属的循环链表中的一个元素   | 而List类型的值则代表一个完整的链表        |
| New函数功能          | 创建并初始化Ring，可以指定包含的元素数量，创建后长度不可变 | 创建并初始化List不能指定包含的元素数量    |
| 初始化               | var r ring.Ring声明的r是一个长度为1的循环链表              | var l list.List声明的l是一个长度为0的链表 |
| 时间复杂度           | Ring的len方法时间复杂度o(N)                                | List的len方法时间复杂度o(1)               |
