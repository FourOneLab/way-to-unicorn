---
title: schema.md
date: 2020-04-14T10:09:14.218627+08:00
draft: false
---

# package schema

```go
import "k8s.io/apimachinery/pkg/runtime/schema"
```

## Variables

```go
var EmptyObjectKind = emptyObjectKind{}
```

EmptyObjectKind将ObjectKind接口实现为noop。

## func ParseKindArg

```go
func ParseKindArg(arg string) (*GroupVersionKind, GroupKind)
```

ParseKindArg接收通用格式的字符串，（如`Kind.group.com`或`Kind.version.group.com`），并将其解析输出。 代码不会知道字符串使用哪种表示形式，但对所有`GroupKinds`都有一定的了解，因此代码可以自己推测。

如果只有两个部分，则`*GroupVersionResource`为`nil`。即`Kind.group.com`解析为：

- `group = com，version = group，kind = Kind`
- `group = group.com，kind = Kind`

## func ParseResourceArg

```go
func ParseResourceArg(arg string) (*GroupVersionResource, GroupResource)
```

ParseResourceArg接收通用格式的字符串，（如`resource.group.com`或`resource.version.group.com`）并将其解析输出。 代码不会知道字符串使用哪种表示形式，但对所有`GroupVersions`都有一定的了解，因此代码可以自己推测。 

如果只有两个段，则`*GroupVersionResource`为`nil`。 即`resource.group.com`解析为：

- `group = com，version = group，resource = resource`
- `group = group.com，resource = resource`

## type GroupKind

```go
type GroupKind struct {
    Group string
    Kind  string
}
```

GroupKind指定Group和Kind，但不强制Version。这对于在查找阶段没有部分有效类型的情况下识别概念很有用。

### func ParseGroupKind

```go
func ParseGroupKind(gk string) GroupKind
```

### func (GroupKind) Empty

```go
func (gk GroupKind) Empty() bool
```

### func (GroupKind) String

```go
func (gk GroupKind) String() string
```

### func (GroupKind) WithVersion

```go
func (gk GroupKind) WithVersion(version string) GroupVersionKind
```

## type GroupResource

```go
type GroupResource struct {
    Group    string
    Resource string
}
```

GroupResource指定一个Group和一个Resource，但不强制Version。这对于在查找阶段没有部分有效类型的情况下识别概念很有用。

### func ParseGroupResource

```go
func ParseGroupResource(gr string) GroupResource
```

ParseGroupResource将`resource.group`字符串转换为GroupResource结构，每个字段都允许使用空字符串。

### func (GroupResource) Empty

```go
func (gr GroupResource) Empty() bool
```

### func (GroupResource) String

```go
func (gr GroupResource) String() string
```

### func (GroupResource) WithVersion

```go
func (gr GroupResource) WithVersion(version string) GroupVersionResource
```

## type GroupVersion

```go
type GroupVersion struct {
    Group   string
    Version string
}
```

GroupVersion包含唯一标识API的“Group”和“Version”。

### func ParseGroupVersion

```go
func ParseGroupVersion(gv string) (GroupVersion, error)
```

ParseGroupVersion将`group/version`字符串转换为GroupVersion结构。如果无法解析字符串，它将报告错误。

### func (GroupVersion) Empty

```go
func (gv GroupVersion) Empty() bool
```

如果Group和Version为空，则Empty返回true。

### func (GroupVersion) Identifier

```go
func (gv GroupVersion) Identifier() string
```

Identifier实现`runtime.GroupVersioner`接口。

### func (GroupVersion) KindForGroupVersionKinds

```go
func (gv GroupVersion) KindForGroupVersionKinds(kinds []GroupVersionKind) (target GroupVersionKind, ok bool)
```

KindForGroupVersionKinds从列表中标识首选GroupVersionKind。 如果没有选项与Group匹配，则返回ok为false。 相比于只有Group匹配，Group与Version都匹配更好。

- TODO：将GroupVersion移至`pkg/runtime`，因为它已被scheme使用
- TODO：在GroupVersion和`runtime.GroupVersioner`之间引入适配器类型，并在较少的地方使用`LegacyCodec(GroupVersion)`

### func (GroupVersion) String

```go
func (gv GroupVersion) String() string
```

String将group和version写入一个单独的`group/version`字符串中，对于旧版v1，将返回v1。

### func (GroupVersion) WithKind

```go
func (gv GroupVersion) WithKind(kind string) GroupVersionKind
```

WithKind基于方法接收者的GroupVersion和传递的Kind创建一个GroupVersionKind。

### func (GroupVersion) WithResource

```go
func (gv GroupVersion) WithResource(resource string) GroupVersionResource
```

WithResource基于方法接收者的GroupVersion和传递的Resource创建一个GroupVersionResource。

## type GroupVersionKind

```go
type GroupVersionKind struct {
    Group   string
    Version string
    Kind    string
}
```

GroupVersionKind明确标识一种kind。它没有匿名包含GroupVersion以避免自动强制。它不使用GroupVersion来避免自定义编组。

### func FromAPIVersionAndKind

```go
func FromAPIVersionAndKind(apiVersion, kind string) GroupVersionKind
```

FromAPIVersionAndKind返回一个GVK，代表不使用TypeMeta提供的字段表示的类型。存在此方法以支持具有不同group和kind的测试类型和旧版序列化。

- DOTO：进一步减少此方法的使用。

### func (GroupVersionKind) Empty

```go
func (gvk GroupVersionKind) Empty() bool
```

如果group，version和kind为空，则Empty返回true。

### func (GroupVersionKind) GroupKind

```go
func (gvk GroupVersionKind) GroupKind() GroupKind
```

### func (GroupVersionKind) GroupVersion

```go
func (gvk GroupVersionKind) GroupVersion() GroupVersion
```

### func (GroupVersionKind) String

```go
func (gvk GroupVersionKind) String() string
```

### func (GroupVersionKind) ToAPIVersionAndKind

```go
func (gvk GroupVersionKind) ToAPIVersionAndKind() (string, string)
```

ToAPIVersionAndKind是一种便捷方法用于满足`runtime.Object`的type而不使用TypeMeta。

## type GroupVersionResource

```go
type GroupVersionResource struct {
    Group    string
    Version  string
    Resource string
}
```

GroupVersionResource明确标识resource。它没有匿名包含GroupVersion以避免自动强制。它不使用GroupVersion来避免自定义编组。

### func (GroupVersionResource) Empty

```go
func (gvr GroupVersionResource) Empty() bool
```

### func (GroupVersionResource) GroupResource

```go
func (gvr GroupVersionResource) GroupResource() GroupResource
```

### func (GroupVersionResource) GroupVersion

```go
func (gvr GroupVersionResource) GroupVersion() GroupVersion
```

### func (GroupVersionResource) String

```go
func (gvr GroupVersionResource) String() string
```

## type GroupVersions

```go
type GroupVersions []GroupVersion
```

GroupVersions可用于表示一组所需的group versions。

- TODO：将GroupVersions移至`pkg/runtime`下，因为它已被scheme使用。
- TODO：在GroupVersion和`runtime.GroupVersioner`之间引入适配器类型，并在较少的地方使用`LegacyCodec(GroupVersion)`

### func (GroupVersions) Identifier

```go
func (gv GroupVersions) Identifier() string
```

Identifier实现runtime.GroupVersioner接口。

### func (GroupVersions) KindForGroupVersionKinds

```go
func (gvs GroupVersions) KindForGroupVersionKinds(kinds []GroupVersionKind) (GroupVersionKind, bool)
```

KindForGroupVersionKinds从列表中标识首选GroupVersionKind。如果没有选项与group匹配，则返回ok为false。

## type ObjectKind

```go
type ObjectKind interface {
    // SetGroupVersionKind设置或清除对象的预期序列化类型。 传递nil将会清除当前设置。
    SetGroupVersionKind(kind GroupVersionKind)
    // GroupVersionKind返回存储的对象的group，version和kind，如果该对象未公开或不提供这些字段，则返回nil。
    GroupVersionKind() GroupVersionKind
}
```

从Scheme序列化的所有对象都对其类型信息进行编码。 序列化使用此接口，以将Scheme中的类型信息设置到对象的序列化版本上。 对于无法序列化或有独特要求的对象，此接口可能是无操作的。
