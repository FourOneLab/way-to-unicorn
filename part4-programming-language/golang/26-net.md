---
title: 26-net
date: 2019-11-25T11:15:47.522182+08:00
draft: false
---

## socket与IPC

socket是一种IPC方法（Inter-Process Communication，进程间通信），IPC主要定义的是多个进程之间，相互通信的方法，这些方法主要包括：

- 系统信号（signal）
- 管道（pipe）
- 套接字（socket）
- 文件锁（file lock）
- 消息队列（message queue）
- 信号灯（semaphore，或称为信号量）等

主流操作系统大都对IPC提供了强力的支持，尤其是socket。

Golang对IPC也提供了一定的支持，如：

- os包和`os/signal`包中针对系统信号的API
- `os.Pipe`函数可以创建命名管道
- `os/exec`包对匿名管道提供支持
- net包对socket提供支持

**在众多的IPC方法中，socket是最为通用和灵活的一种**，与其他的IPC方法不同，利用socket进行通信的进程，可以不局限于同一台计算机当中。通信双方只要能够通过计算机的网卡端口以及网络进行通信，就可以使用socket。

支持socket的操作系统一般都会对外提供一套API。**跑在它们之上的应用程序利用这套API，就可以与互联网上的另一台计算机中的程序、同一台计算机中的其他程序，甚至同一个程序中的其他线程进行通信**。

Linux操作系统中，用于创建socket实例的API，就是一个名为socket的系统调用，这个系统调用是Linux内核的一部分。

> 所谓系统调用，可以理解为特殊的C语言函数，它们是连接应用程序和操作系统内核的桥梁，也是应用程序使用操作系统功能的**唯一渠道**。

- syscall包中有一个与socket系统调用相对应的函数，这两者的函数签名基本一致，都会接收三个int类型的参数，并会返回一个可以代表文件描述符的结果。

    ```go
    func Socket(domain, typ, proto int) (fd int, err error)
    ```

syscall包中的Socket函数本身是与平台不相关的，在其底层，Go语言为它支持的每个操作系统都做了适配，这才使得这个函数无论在哪个平台上，总是有效的。

- net包中的很多程序实体都会直接或间接地使用`syscall.Socket`函数
- 调用`net.Dial`函数的时候会为它的两个参数设定值，第一个参数名为network，它决定了Go程序底层会创建什么样的socket实例，并使用什么样的协议与其他程序通信   

### `net.Dial`函数的一个参数network的可选值

```go
func Dial(network, address string) (Conn, error)
```

`net.Dial`函数接受两个参数，network和address，都是string类型的。

network参数常用的可选值有9个，这些值分别代表了socket实例可使用的不同通信协议：

- tcp：代表TCP协议，其基于的IP协议的版本根据参数address的值自适应
- tcp4：代表基于IP协议第四版的TCP协议
- tcp6：代表基于IP协议第六版的TCP协议
- udp：代表UDP协议，其基于的IP协议的版本根据参数address的值自适应
- udp4：代表基于IP协议第四版的UDP协议
- udp6：代表基于IP协议第六版的UDP协议
- unix：代表Unix通信域下的一种内部socket协议，以SOCK_STREAM为socket类型
- unixgram：代表Unix通信域下的一种内部socket协议，以SOCK_DGRAM为socket类型
- unixpacket：代表Unix通信域下的一种内部socket协议，以SOCK_SEQPACKET为socket类型

### `syscall.Socket`函数接受的三个参数

```go
func Socket(domain, typ, proto int) (fd int, err error)
```

这三个参数都是int类型，这些参数代表分别是：

domain：socket通信域，主要有：

- IPv4域：基于IP协议第四版的网络（syscall中的常量AF_INET表示）
- IPv6域：基于IP协议第四版的网络（syscall中的常量AF_INET6表示）
- Unix域：一种类Unix操作系统中特有的通信域，装有此类操作系统的同一台计算机中，应用程序可以基于此域创建socket连接（syscall中的常量AF_UNIX表示）

typ：类型，共有四种:

- SOCK_DGRAM：代表datagram即数据报文，一种**有消息边界**，但没**有逻辑连接**的**非可靠**socket类型，基于UDP协议的网络通信属于此类。

    > 有消息边界指的是，与socket相关的操作系统内核中的程序在发送或接收数据的时候是以消息为单位的。把消息理解为带有固定边界的一段数据，内核程序自动识别和维护这种边界，在必要的时候，把数据切割成一个个消息，或者把多个消息串接成连续的数据，这样应用程序只需要面向消息进处理就可以了。

    > 有逻辑连接指的是，通信双发在收发数据之前必须先建立网络连接，待连接建立好之后，双方就可以一对一地进行数据传输，基于UDP协议的网络通信是没有逻辑连接的。只要应用程序指定好对方的网络地址，内核程序就可以立即把数据报文发送出去。

    > 优势：发送速度快，不长期占用网络资源，并且每次发送都可以指定不同的网络地址。

    > 劣势：每次都需要指定网络地址使得数据报文更长，无法保证传输的可靠性，不能实现数据的有序性，数据只能单向进行传输。

- SOCK_STREAM：与SOCK_DGRAM相反，它是**没有消息边界**，但**有逻辑连接**，能够保证传输的**可靠性**和数据的**有序性**，同时还可以实现数据的**双向传输**。基于TCP协议的网络通信属于此类。

    > 这样的网络通信传输数据的形式是**字节流**（字节流是以字节为单位的），而不是数据报文。内核程序无法感知一段字节流中包含了多少个消息，以及这些消息是否完整，这完全需要应用程序自己把控。

    > 此类网络通信中的一端，总是会忠实地按照另一端发死你个数据时的字节排序，接收和缓存它们，所以应用程序需要更具双方的约定去数据中查找消息边界，并按照边界切割数据。

- SOCK_SEQPACKET

- SOCK_RAW

syscall包中都有同名常量与之对应。

proto：协议；表示socket实例所使用的协议，通常明确了前两个参数，就不在需要确定第三个参数值了，一般设置为0即可，内核程序会自动选择最适合的协议。

- 当两个参数分别为`syscall.AF_INET`和`syscall.SOCK_DGRAM`的时候，内核程序会选择UDP作为协议
- 当两个参数分别为`syscall.AF_INET6`和`syscall.SOCK_STREAM`的时候，内核程序会选择TCP作为协议

![images](/images/syscall-socket.png)

在使用net包中的高层次API的时候，前两个参数（domain和typ）也不需要给定，只需要把前面罗列的9个可选值字符串字面量的其中一个，作为network参数的值就好了。

## 调用`net.DialTimeout`函数时设定超时时间

```go
func DialTimeout(network, address string, timeout time.Duration) (Conn, error)
```

超时时间代表这函数为网络连接建立完成而等待的最长时间，**这是一个相对时间**，由函数的参数timeout的值表示。

开始的时间点几乎是调用`net.DialTimeout`函数的那一刻，之后的时间，主要会花费在：

- 解析参数network和address的值：在这个过程中，函数会确定网络服务的IP地址、端口号等必要信息、并在需要时访问DNS服务。

> 如果解析出的IP地址有多个，那么函数会串行或并发地尝试建立连接，无论以什么方式尝试，函数总会以最先建立成功的那个连接为准。**会根据超时前剩余的时间，去设定每次连接尝试的超时时间，以便让它们都有适当的时间执行**。

- 创建socket实例并建立网络连接。

不论执行到哪一步，只要绝对的超时时间到达的那一刻，网络连接还没有建立完成，该函数就会返回一个代表I/O操作超时的错误值。

> net包中有一个名为Dialer的结构体类型，该类型有一个Timeout字段，与上述timeout参数的含义完全一致，实际上，`net.DialTimeout`函数正是利用了这个类型的值才得以实现功能的。

## `net/http`

使用`net.Dial`和`net.DialTimeout`函数访问基于HTTP协议的网络服务是完全没有问题的，HTTP协议是基于TCP/IP协议栈，并且是一个面向普通文本的协议。如果需要方便的访问基于HTTP协议的网络服务，则使用`net/http`包。其中最便捷的是使用`http.Get`函数，在调用它的时候只需要传入一个URL就可以，如下所示：

```go
url1 := "http://google.cn"
fmt.Printf("Send request to %q with method GET ...\n", url1)
resp1, err := http.Get(url1)
if err != nil {
    fmt.Printf("request sending error: %v\n", err)
}
defer resp1.Body.Close()
line1 := resp1.Proto + " " + resp1.Status
fmt.Printf("The first line of response:\n%s\n", line1)
```

```go
func (c *Client) Get(url string) (resp *Response, err error)
```
`http.Get`函数返回两个结果值：

- 第一个结果值的类型是`*http.Response`，它是网络服务给我们传回的函数内容的结构化表示
- 第二个结果值是error类型，它代表了在创建和发送HTTP请求，以及接收和解析HTTP相应的过程中可能发生的错误

`http.Get`函数会在内部使用缺省的HTTP客户端，并且调用它的Get方法以完成功能，这个缺省的HTTP客户端由`net/http`包中的公开变量DefaultClient代表，其类型是`*http.Client`，它的基本类型也是可以被拿来使用的，甚至它是开箱即用的，如下代码所示：

```go
var httpClient1 http.Client
resp2, err := httpClient1.Get(url1)

// 等价于
resp1, err := http.Get(url1)
```

`http.Client`是一个结构体类型，并且它包含的字段都是公开的。之所以该类型的零值仍然可用，是因为它的这些字段要么存在着相应的缺省值，要么其零值直接可以使用，且代表着特定的含义。

### `http.Client`类型中Transport字段的含义

`http.Client`类型中的Transport字段代表着：向网络服务发送HTTP请求，并从网络服务接收HTTP相应的操作过程。也就是说，该字段的方法RoundTrip应该实现单次HTTP事务（或者说基于HTTP协议的单次交互）需要的所有步骤。

这个字段是`http.RoundTripper`接口类型的，它有一个由`http.DefaultTransport`变量代表的缺省值，当我们在初始化一个`http.Client`类型的值的时候，如果显式地为该字段赋值，那么这个Client值就会直接使用`http.DefaultTransport`。

#### DefaultTransport

DefaultTransport的实际类型是`*http.Transport`，后者即为`http.RoundTripper`接口的默认实现，这个类型是可以被复用的，它是并发安全的，因此`http.Client`类型也拥有同样的特质。

`http.Transport`类型会在内部使用一个`net.Dialer`类型的值，并且会把该值的Timeout字段的值，设定为30秒。如果30秒之内没有建立好连接就会判断为操作超时。在DefaultTransport的值被初始化的时候，这样的`net.Dialer`值的DialContext方法会被赋给前者的DialContext字段。

`http.Transport`类型还包含很多其他的字段，其中有一些字段是关于操作超时的：

- IdleConnTimeout：空闲的连接在多久之后就应该被关闭。

    > DefaultTransport会把该字段的值设置为90秒，如果该值设置为0 ，那么就表示不关闭空闲的连接。**这样会造成资源的泄露**。

    与该字段相关的一些字段：

    - MaxIdleConns：无论当前的`http.Transport`类型的值访问了多少个网络服务，这个字段都只会对空闲连接的总数做出限定。
    - MaxIdleConnsPerHost：这个字段限定的是`http.Transport`值访问的每一个网络服务的最大空闲连接数。

        每个网络服务都会有自己的网络地址，可能会使用不同的网络协议，对一些HTTP请求也可能会使用代理，`http.Transport`值就是通过这三方面的具体情况，来鉴别不同的网络服务的。MaxIdleConnsPerHost字段的缺省值，由`http.DefaultMaxIdleConnsPerHost`变量代表，值为2。即，在默认情况下，对某个`http.Transport`值访问的每一个网络服务，它的空闲连接数最多只能有两个。

    - MaxConnsPerHost：针对某个`http.Transport`值访问的每一个网络服务的最大连接数，不论这些连接是否空闲的，该字段没有相应的缺省值，它的零值表示不对此设限制。

- ResponseHeaderTimeout：从客户端把请求完全递交给操作系统到从操作系统那里接收到响应报文头的最大时长。

    > DefaultTransport没有设定该字段的值。

- ExpectContinueTimeout：在客户端递交了请求报文头之后，等待接收第一个响应报文头的最长时间。

    > 在客户端想要使用HTTP的POST方法把一个很大的报文体发送给服务端的时候，它可以先通过发送一个包含了Expect:100-continue的请求报文头，来询问服务端是否愿意接收这个大报文体。

    这个字段是用于设定在这种情况下的超时时间的，注意，如果该字段的值不大于0 ，那么无论多大的请求报文都将会立即发送出去，这可能会造成网络资源的浪费。DefaultTransport把该字段的值设定为1秒。

- TLSHandshakeTimeout：（TLS是Transport Layer Security的缩写，翻译为传输层安全），这个字段代表了基于TLS协议的连接在被建立时的握手阶段的超时时间。若该值为0 ，则表示对这个时间不设限。

    > DefaultTransport把该字段的值设定为10秒。

产生空闲连接的原因：HTTP协议有一个请求报文头叫做“Connection”，在HTTP 1.1 中，这个报文头的值默认是“keep-alive”，在这种情况下的网络连接都是持久连接，它们会在当前的HTTP事物完成后仍然保持着连通性，因此是可以被复用的，那就有两种可能：

1. 针对同一个网络服务，有新的HTTP请求被递交，该连接被再次使用
2. 不再有针对该网络服务的HTTP请求，该连接被闲置（这就会产生空闲的连接）

如果分配给某个网络服务的连接过多的话，也可能会导致空闲连接的产生，因为每一个新递交的HTTP请求，都只会征用一个空闲的连接，所以为空闲连接设定限制，在大多数情况下是很有必要的。

> 如果要杜绝空连接产生，可以在初始化`http.Transport`值的时候把它的DisableKeepAlives字段的值设置为true，这时HTTP请求的“Connection”报文头的值就会被设置为“close”，这会告诉网络服务，这个网络连接不必保持，当前的HTTP事物完成后就可以断开它了。这样每一个HTTP请求被递交时，就会产生一个新的网络连接，明显加重网络服务以及客户端的负载，并会让每个HTTP事物都消耗更多的时间。一般情况下不会设置DisableKeepAlives。

在`net.Dialer`类型中也有一个keepAlive字段，它是直接作用在底层的socket上的，一种针对网络连接（TCP连接）的存活探测机制。它的值用于表示每隔多长时间发送一次探测包，当该值不大于0是，则表示不开启这个机制。DefaultTransport会把这个字段的值设定为30秒。

### `http.Server`类型的ListenAndServer方法

`http.Server`代表的是基于HTTP协议的网络服务，它的ListenAndServer方法的功能是：监听一个基于TCP协议的网络地址，并对接收到的HTTP请求进行处理。这个方法会默认开启针对网络连接的存活探测机制，以保证连接是持久的。该方法会一直执行，直到有严重的错误发送或者被外界关闭。当被外界关闭时，会返回一个由`http.ErrServerClosed`变量代表的错误值。

ListenAndServer方法主要做下面几件事情：

1. 检查当前的`http.Server`类型的值的Addr字段，该字段的值代表了当前的网络服务需要使用的网络地址，即IP地址和端口号。

    > 如果该字段的值为空字符串，那么就用“:http代替，也就是使用任何可以代表本机的域名和IP地址，并且端口号为80。

2. 通过调用`net.Listen`函数在已确定的网络地址上启动基于TCP协议的监听。
3. 检查`net.Listen`函数返回的错误值，如果该错误值不是nil，那么直接返回该值，否则通过调用当前值的Serve方法准备接收和处理将要到来的HTTP请求。
   1. `net.Listen`函数完成如下操作：
      1. 解析参数值中包含的网络地址隐含的IP地址和端口号
      2. 根据给定的网络协议，确定监听的方法，并开始进行监听
