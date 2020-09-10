---
title: package runtime
date: 2020-04-14T10:09:14.218627+08:00
draft: false
---

```go
import "k8s.io/apimachinery/pkg/runtime"
```

runtime定义了通用类型和结构之间的转换，以将查询字符串映射到结构对象。

runtime包括辅助函数，该函数用于遵循kubernetes API对象约定的API对象，这些约定是：

1. API对象具有一个通用的元数据结构成员TypeMeta。
2. 代码引用了一组内部API对象。
3. 在单独的包中，具有一组外部API对象。
4. 外部API对象组被认为是版本化的，并且从未对其进行过任何重大更改（可以添加字段，但不能更改或删除字段）。
5. 随着API的发展，将在每次重大更改后制作一个附加的版本包。
6. Versioned包具有转换功能，可以与内部版本进行转换。
7. 将根据弃用策略继续支持较旧的版本，并且由于第六个原因，可以轻松提供`program/library`以将旧版本更新为新版本。
8. 所有的序列化和反序列化都在一个集中的位置进行处理。

runtime提供一个转换辅助工具使得第六个变得很容易，并且`Encode/Decode/DecodeInto`完成了第八个。还可以注册使用对应版本的其他“编解码器”。建议在程序包的`init`函数中向`runtime`注册类型。

另外，`types.go`中提供了一些适用于所有API对象和版本的常见类型。
