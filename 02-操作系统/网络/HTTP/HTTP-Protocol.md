---
title: HTTP-Protocol
date: 2020-04-14T10:09:14.242627+08:00
draft: false
---

- [0.1. HTTP/1.1 首部字段](#01-http11-首部字段)
  - [0.1.1. 通用首部字段](#011-通用首部字段)
  - [0.1.2. 实体首部字段](#012-实体首部字段)
- [0.2. HTTP请求](#02-http请求)
  - [0.2.1. 请求方法](#021-请求方法)
    - [0.2.1.1. 浏览器对请求方法的支持](#0211-浏览器对请求方法的支持)
  - [0.2.2. 请求首部](#022-请求首部)
- [0.3. HTTP响应](#03-http响应)
  - [0.3.1. 响应状态码](#031-响应状态码)
  - [0.3.2. 响应首部](#032-响应首部)
- [0.4. URI](#04-uri)

## 0.1. HTTP/1.1 首部字段

HTTP首部字段类型：

1. 通用首部字段（General Header Fields）
2. 请求首部字段（Request Header Fields）
3. 响应首部字段（Response Header Fields）
4. 实体首部字段（Entity Header Fields）

### 0.1.1. 通用首部字段

|首部字段|作用描述|
|---|---|
|Cache-Control|控制缓存行为|
|Connection|逐跳首部、连接的管理|
|Date|创建报文的日期时间|
|Pragma|报文指令|
|Trailer|报文末端的首部一览|
|Transfer-Encoding|指定报文主体的传输编码方式|
|Upgrade|升级为其他协议|
|Via|代理服务器的相关信息|
|Warning|错误通知|

### 0.1.2. 实体首部字段

|首部字段|作用描述|
|---|---|
|Allow|资源可支持的HTTP方法
|Content-Encoding|实体主体适用的编码方式
|Content-Language|实体主体的自然语言
|Content-Length|实体主体的大小（单位：字节）
|Content-Location|替代对应资源的URI
|Content-MD5|实体主体的报文摘要
|Content-Range|实体主体的位置范围
|Content-Type|实体主体的媒体类型
|Expires|实体主体的过期时间
|Last-Modified|资源的最后修改时间

## 0.2. HTTP请求

HTTP是一种请求-相应协议，协议涉及的所有事情都是以一个请求开始。HTTP请求跟其他所有HTTP报文一样，都是由一系列文本行组成，这些文本行会按照以下顺序进行排列：

1. 请求行（request-line）
2. 零个或任意多个请求首部（header）
3. 一个空行
4. 可选的报文主体（body）

一个典型的HTTP请求，如下所示：

```http
GET /Protocols/rfc2616/rfc2616.html HTTP/1.1    // 请求行
Host: www.w3.org    // 请求首部
User-Agent: Mozilla/5.0     // 请求首部
(empty line)    // 空行，必须存在
// 可选报文主体部分为空
```

- 请求行：
  - 第一个单词是请求方法（request method）
  - 之后是统一资源标识符（Uniform Resource Identifier，URI）
  - 以及所使用的HTTP版本

- 可选的报文主体：报文是否包含主体需要根据请求使用的方法而定

### 0.2.1. 请求方法

请求方法是请求行中的第一个单词，指明了客户端想要对资源执行的操作。

- HTTP0.9：只有GET一个方法
- HTTP1.0：添加了POST和HEAD方法
- HTTP1.1：添加了PUT、DELETE、OPTIONS、TRACE和CONNECT，允许开发着自行添加更多方法

> HTTP1.1要去必须实现的只有GET和HEAD方法，其他方法的实现是可选的。

各个HTTP方法的作用说明：

| 方法|描述|说明|是否安全|是否幂等|
|---|---|---|---|---|
|GET|命令服务器返回指定的资源||是
|HEAD|与GET方法类似，唯一的不同在于这个方法不要求服务器返回报文的主体|通常用于在不获取报文主体的情况下，获取相应的首部|是
|POST|命令服务器将报文主体中的数据传递给URI指定的资源，至于服务器具体会对这些数据执行什么动作取决于服务器本身||不是
|PUT|命令服务器将报文主体中的数据设置为URI指定的资源|如果URI指定的位置上已经有数据存在，那么使用报文主体的数据去替代已有的数据,如果资源尚未存在，那么在URI指定的位置上新创建一个资源|不是|幂等
|DELETE|命令服务器删除URI指定的资源||不是|幂等
|TRACE|命令服务器返回请求本身|通过这个方法，客户端可以知道介于它和服务器之间的其他服务器是如何处理请求的|是
|OPTIONS|命令服务器返回它支持的HTTP方法列表||是
|CONNECT|命令服务器与客户端建立一个网络连接，这个方法通常用于设置SSL隧道已开启HTTPS功能
|PATCH|命令服务器使用报文主体中的数据对URI指定的资源进行修改

> 安全的方法：如果一个HTTP方法只要求服务器提供信息而不会对服务器的状态做任何修改，那么这个方法是安全的。
> 幂等的方法：如果一个HTTP方法在使用相同的数据进行第二次调用的时候，不会对服务器的状态造成任何改变，那么这个方法是幂等的。

#### 0.2.1.1. 浏览器对请求方法的支持

- GET方法是最基本的HTTP方法，它负责从服务器获取内容，所有浏览器都支持这个方法。
- POST方法从HTML2.0开始可以通过HTML表单来实现：HTML的form标签有一个名为method的属性，用户可以通过将这个属性的值设置为get或者post来指定要使用那种方法。

> HTML不支持除了GET和POST之外的其他HTTP方法。

主流的浏览器通常都不会只支持HTML一种数据格式，用户可以使用XMLHttpREquest（XHR）来获得对PUT方法和DELETE方法的支持。

> XHR是一系列浏览器API，这些API同化成那个有JS包裹（实际上XHR就是一个名为XML
XMLHttpRequest的浏览器对象），XHR允许程序员向服务器发送HTTP请求，这项技术不仅仅局限于XML格式，包括JSON以及纯文本在内的任何格式的请求和相应都可以通过XHR发送。

### 0.2.2. 请求首部

HTTP请求方法定义了发送请求的客户端想要执行的动作，而HTTP请求的首部则记录了与请求本身以及客户端有关的信息。请求的首部由任意多个用冒号分隔的纯文本键值对组成，最后以回车（CR）和换行（LF）结尾。

大多数HTTP请求首部都是可选的，Host首部字段是HTTP1.1唯一强制要求的首部。根据请求使用的方法不同，如果请求的报文中包含有可选的主体，那么请求的首部还需要带有内容长度（Content-Length）字段或者传输编码（Transfer-Encoding）字段。

常见请求首部如下：

|首部字段|作用描述|说明|
|---|---|---|
|Accept|客户端在HTTP响应中能够接收的内容类型|比如，客户端可以通过`Accept:text/html`这个首部，告知服务器自己希望在相应的主体中收到HTML类型的内容
|Accept-Charset|客户端要求服务器使用的字符集编码|比如，客户端可以通过`Accept-Charset:utf-8`这个首部来告知服务器自己希望响应的主体使用UTF-8字符集
|Accept-Encoding|优先的内容编码
|Accept-Language|优先的语言（自然语言）
|Authorization|这个首部用于向服务器发送基本的身份验证证书
|Cookie|客户端应该在这个首部中把服务器之前设置的所以Cookie回传给服务器|比如，如果服务器之前在浏览器上设置了3个Cookie，那么Cookie首部字段将在一个字符串里面包含这3个Cookie，并使用分号对这些Cookie进行分隔，如`Cookie:my_first_cookie=hello;my_second_cookie=world`
|Content-Length|请求主体的字节长度
|Content-Type|当请求包含主体的时候，这个首部用于记录主体内容的类型|在发送POST或PUT请求时，内容的类型默认为`x-www-form-urlen-coded`，在上传文件时，内容的类型应该设置为`multipart/form-data`（上传文件这一操作可以通过将input标签的类型设置为file来实现）
|Expect|期待服务器的特定行为
|From|用户的电子邮箱地址
|Host|服务器的名字以及端口号|如果这个首部没有记录服务器的端口号，就表示服务器使用的是80端口
|if-Match|比较实体标记（ETag）|
|if-None-Match|比较实体标记（与if-Match相反）|
|if-Modified-Since|比较资源的更新时间|
|if-Unmodified-Since|比较资源的更新时间（与if-Modified-Since相反）|
|if-Range|资源未更新时发送实体Bytes的范围请求|
|Max-Forwards|最大传输逐跳数|
|Proxy-Authorization|代理服务器要求客户端的认证信息|
|Range|实体的字节范围请求|
|Referrer|发起请求的页面所在地址
|TE|传输编码的优先级|
|User-Agent|对发起请求的客户端进行描述

## 0.3. HTTP响应

HTTP响应报文是对HTTP请求报文的回复。跟HTTP请求一样，HTTP相应也是有一系列文本行组成的，其中包括：

- 一个状态行
- 零个或任意数量的响应首部
- 一个空行
- 一个可选的报文主体

HTTP响应，如下：

```http
200 OK  // 状态行，包含状态码和响应的原因短语（对状态码进行简单的描述）
Date: Sat, 22 Nov 2014 12:58:58 GMT
Server: Apache/2
　Last-Modified: Thu, 28 Aug 2014 21:01:33 GMT
Content-Length: 33115
Content-Type: text/html; charset=iso-8859-1

// HTML格式的报文主体
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/
　　 TR/xhtml1/DTD/xhtml1-strict.dtd"> <html xmlns='http://www.w3.org/1999/
　　 xhtml'> <head><title>Hypertext Transfer Protocol -- HTTP/1.1</title></
　　 head><body>...</body></html>
```

### 0.3.1. 响应状态码

HTTP响应中的状态码表明了响应的类型，HTTP响应状态码共有5种类型，分别以不同的数字作为前缀。

| 状态码类型|作用描述|说明|
|---|---|---|
|1XX|情报状态码|服务器通过这些状态码来告知客户端，自己已经接收到了客户端发送的请求，并且已经对请求进行了处理
|2XX|成功状态码|这些状态码说明服务器已经接收到了客户端的请求，并且成功地对请求进行了处理，这类状态码的标准响应为“200 OK”
|3XX|重定向状态码|这些状态码表示服务器已经收到了客户端发送的请求，并且已经成功处理了请求，但是为了完成请求指定的动作，客户端还需要再做一些其他的工作，这里状态码大多用于实现URL重定向。
|4XX|客户端错误状态码|这类状态码说明客户端发送的请求出现了某些问题。在这一类型的状态码中，最常见的就是“404 Not Found”，这个状态码表示服务器无法从请求指定的URL中找到客户端想要的资源
|5XX|服务器错误状态码|当服务器因为某些原因而无法正确地处理请求时，服务器就会使用这类状态码来通知客户端，这一类型的状态码中，最常见的就是“500 Internal Server Error”状态码。

### 0.3.2. 响应首部

响应首部跟请求首部一样，是由冒号分隔的纯文本键值对组成，并且同时以回车（CR）和换行（LF）结尾。

- 请求首部能够告诉服务器更多与请求相关或者与客户端诉求相关的信息
- 响应首部能够向客户端传达更多响应相关或者与服务器诉求相关的信息

常见响应首部如下：

|首部字段|作用描述|
|---|---|
|Accept-Ranges|是否接收字节范围请求
|Age|推算资源创建经过时间
|Allow|告知客户端，服务器支持那些请求方法
|Content-Length|响应主体的字节长度
|Content-Type|如果响应包含可选的主体，那么这个首部记录的就是主体的内容的类型
Date|以格林尼治标准时间（GMT）格式记录的当前时间
|ETag|资源的匹配信息|
|Location|这个首部仅在重定向的时候使用，它会告诉客户端接下来应该向哪个URL发送请求
|Proxy-Authenticate|代理服务器对客户端的认证信息
|Retry-After|对再次发起请求的时机要求
|Server|响应的服务器的域名
|Set-Cookie|在客户端里设置一个Cookie，一个响应里面可以包含多个Set-Cookie首部
|Vary|代理服务器缓存的管理信息|
|WWW-Authenticate|服务器通过这个首部来告诉客户端，在Authorization请求首部中应该提供哪种类型的身份验证信息。服务器异常会把这个首部与“401 Unauthorized”状态行一同发送。除此之外，这个首部还会向服务器许可的认证授权模式（scheme）提供验证信息（challenge information）

## 0.4. URI

- 一种使用字符串表示资源名的方法：统一资源名称（Uniform Resource Name，URN）
- 一种使用字符串表示资源所在为止的方法：统一资源定位符（Uniform Resource Location，URL）

> URI是一个涵盖性术语，它包含了URN和URL，并且这两者也拥有相似的语法和格式。

URI一般格式为：`<方案名称>:<分层部分>[?<查询参数>][#<片段>]`

URI的方案名称（scheme name）记录了URI正在使用的方案，它定义了URI其余部分的结构。因为URI是一种非常常用的资源标识方式，所以它拥有大量的方案可供使用，大部分时候使用HTTP方案。

URI的分层部分（hierarchical part）包含了资源的识别信息，这些信息以分层的方式进行组织。如果分层部分以双斜杠（//）开头，那么说明它包含了可选的用户信息，这些信息将以@符合结尾，后跟分层路径。不带用户信息的分层部分就是一个单纯的路径，每个路径都由一连串的分段（segment）组成，各个分段之间使用单斜线（/）分隔。

在URI的各个部分当中，只有“方案名称”和“分层部分”是必须的。以问号（?）为前缀的查询参数（query）是可选的，这些参数用于包含无法使用分层方式表示的其他信息。多个查询参数会被组织成一连串的键值对，，各个键值对之间使用&符号分隔。

URI的另一个可选部分为片段（fragment）,片段使用井号（#）作为前缀，它可以对URI定义的资源中的次级资源进行表示（secondary resource）进行标识。当URI包含参数时，URI的片段将被放在查询参数之后。因为URI的片段是有客户端负责处理的，所以Web浏览器在将URI发送给服务器之前，一般都会先把URI中的片段移除。（如果想要取得URI片段，那么可以通过JS或某个HTTP客户端库，将URI片段包含在一个GET请求里面）。

使用HTTP方案的URI示例：`http://sausheong:password@www.example.com/docs/file?name=sausheong&location=singapore#summary`

- 这个URI使用的是HTTP方案
- 跟在方案之后的是一个冒号，位于@符号之前的分段`sausheong:password`
- 跟在用户信息之后的是`www.example.com/docs/file`是分层部分的其余部分，位于分层部分最高层的是服务器的域名`wwwexample.com`，之后跟着的两个层分别为doc和file。每个分层用单斜杠分隔。
- 跟在分层部分之后是以问号（?）为前缀的查询参数，这个部分包含了`name=sausheong`和`location=singapore`这两个键值对，键值对之间使用&符号连接
- 最后，这个URI的末尾还带有一个以井号（#）前缀的片段

因为每个URI都是一个单独的字符串，所以URI里面是不能够包含空格的，此外，因为问号（?）和井号（#）等符号在URL中具有特殊的含义，所以这些符号是不能够用于其他用途的，未来避开这些限制，需要使用URI编码（百分号编码）来对特殊符号进行转换。

> 所有保留字符需要进行百分号编码，把保留字符转换成该字符在ASCII编码中对应的字节值（byte value）,接着把这个字节值表示为一个两位长的十六进制数组，最后再在这个十六进制数字的前面加上一个百分号（%）。

例如，空格在ASCII编码中的字节值为32，即十六进制中的20。因此，经过百分号编码处理的空格就成了%20，URI中所有空格都会被替换成这个值，如下所示：`http://www.example.com/docs/file? name=sau%20sheong&location=singapore`。
