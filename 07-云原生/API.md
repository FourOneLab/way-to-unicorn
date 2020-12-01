# API

## k8s.io

### apimachinery

[README](https://pkg.go.dev/k8s.io/apimachinery?readme=expanded#section-readme)

提供 Kubernetes 或类 Kubernetes API 对象的 Scheme, typing, encoding, decoding 和 conversion 的库。

这个库是客户端和服务端共享依赖的，用于和 Kubernetes API 基础实施交互而不直接依赖类型。这个库最直接的使用者是 `k8s.io/kubernetes`, `k8s.io/client-go`, 和 `k8s.io/apiserver`。

#### runtime

[README](https://pkg.go.dev/k8s.io/apimachinery@v0.19.4/pkg/runtime)

定义通用类型和结构体之间的转换，来把查询字符串映射到结构对象。

这个库中提供一些辅助函数，用于操作符合 Kubernetes API 对象规范的自定义 API 对象。所以编写的代码要符合如下约定：

0. 自定义的 API 对象有一个通用的元数据字段 `runtime.TypeMeta`
1. 自定义的 API 对象引用一组 Kubernetes 内部的 API 对象
2. 在单独的包中有一组外部的 API 对象
3. 这组外部 API 对象是版本化的，并且没有重大的变更（可以添加新字段，但是没有字段修改或删除）
4. 随着 API 的演化，在每次重大变更时需要为自定义的 API 对象新建一个单独的版本化的包
5. 版本化的包中需要有转换函数，用于自定义 API 对象和 Kubernetes 内部 API 对象之间的相互转换
6. 需要根据弃用策略来继续支持旧版本，因为符合约定5，所以可以很容易的提供库来把旧版本更新到新版本
7. 所有的序列化和反序列化都要在一个集中的地方处理

runtime 包提供一个辅助转换函数，这使得自定义 API 对象很容易满足约定5，同时提供了`Encode` `Decode` `DecodeInto`来满足约定7。也可以注册其他的编解码器，推荐在代码包的 `init` 函数中向 runtime 注册自定义的 types（API对象）。

#### apis/meta/v1

> 未来大概率会将这些不同分类的对象迁移到独立的代码包中。

包含所有版本通用的 API 类型。分为两种类型：

- 外部（序列化）类型：没有自己的版本，如 TypeMeta
- 内部（不序列化）类型：被多种不同的 API 组依赖，放在这里避免重复或循环导入，如标签选择器（LabelSelector）