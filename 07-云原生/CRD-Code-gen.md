# Code Gen for CRD

## 生成控制器代码

`client-go` 要求 `runtime.Object` 类型必须有一个 `DeepCopy()` 方法。也就是说，Go语言实现的CR必须实现 `runtime.Object` 接口。

```go
package runtime
// Object interface must be supported by all API types registered with Scheme. Since objects in a scheme are
// expected to be serialized to the wire, the interface an Object must provide to the Scheme allows
// serializers to set the kind, version, and group the object is represented as. An Object may choose
// to return a no-op ObjectKindAccessor in cases where it is not expected to be serialized.
type Object interface {
	GetObjectKind() schema.ObjectKind
	DeepCopyObject() Object
}

package schema
// All objects that are serialized from a Scheme encode their type information. This interface is used
// by serialization to set type information from the Scheme onto the serialized version of an object.
// For objects that cannot be serialized or have unique requirements, this interface may be a no-op.
type ObjectKind interface {
	// SetGroupVersionKind sets or clears the intended serialized kind of an object. Passing kind nil
	// should clear the current setting.
	SetGroupVersionKind(kind GroupVersionKind)
	// GroupVersionKind returns the stored group, version, and kind of an object, or an empty struct
	// if the object does not expose or provide these fields.
	GroupVersionKind() GroupVersionKind
}
```

代码生成器 [code-generator](https://github.com/kubernetes/code-generator) 主要提供如下功能：

| 代码生成器 | 功能 | 描述 |
| --- | --- |
| client-gen | 为 CR 的 API 组创建类型化的客户端集合 | 实现功能齐全、生产可用 controller 的基础，与 k8s 内置 controller 的实现机制及使用的标准库一致 |
| conversion-gen | 在创建聚合API服务器时，用于实现内外部类型之间的转换功能 |
| deepcopy-gen | 为每一种类型提供一个 `DeepCopy()` 方法，`func(t* T) DeepCopy() *T` | 实现功能齐全、生产可用 controller 的基础 |
| defaulter-gen | 用于生成一些默认的字段 |
| go-to-protobuf |
| import-boss |
| informer-gen | 为 CR 创建 informer，用途是基于事件接口，对服务器上 CR 的变更作出响应 | 实现功能齐全、生产可用 controller 的基础 |
| lister-gen | 为 CR 创建 lister ，用途是对 GET 和 LIST 请求提供一个只读的缓存层 | 实现功能齐全、生产可用 controller 的基础 |
| openapi-gen |
| prerelease-lifecycle-gen |
| register-gen |
| set-gen |

官方提供了一个使用代码生成器生成 controller 的示例仓库：[sample-controller](https://github.com/kubernetes/sample-controller)。

### 使用

所有的 Kubernetes 代码生成器都是在 [gengo](https://github.com/kubernetes/gengo) 的基础上创建的，它们共享很多公用的命令行标识。通常，所有的生成器都是从 `--input-dirs` 中获取输入，然后一个类型一个类型的转换为生成的代码。

- 生成代码与输入代码在同一路径：像 `deepcopy-gen` 那样通过 `--output-file-base "zz_generated.deepcopy"` 来定义输出的文件名。
- 生成代码保存在一个或多个包中：使用 `--output-package` 来指定，输出类似 `client-`，`informer-`，就像 `lister-gen` 那样通常输出在 `pkg/client`。

代码生成器提供来一个 [generate-groups.sh](https://github.com/kubernetes/code-generator/blob/master/generate-groups.sh) 脚本，这样就不需要熟悉那么多的命令行参数。

1. 生成脚本保存在 `hack/update-codegen.sh` ，生成的所有 API 都在 `pkg/apis`，所有 clientsets、informers、listers 都在 `pkg/client` 中。
2. 验证脚本保存在 `hack/verify-codegen.sh` ，如果生成的文件不是最新的，那么执行后会返回一个非零的输出，这在 CI/CD 中很有用。

### 标签

代码生成器的一些行为可以通过命令行参数来控制，但是更多的属性通过源文件中的标签来控制。有两种类型的标签：

- 全局标签，作用在 `package` 上，通常单独创建一个 `doc.go` 文件来存放全局标签
- 局部标签，作用在 `type` 上，在指定的类型定义的源代码上方，用空行隔开

通常标签看起来是这样的：`// +tag-name` 或者 `// +tag-name=value`，都是写在注释中的，因此注释的位置就显得很重要了。大部分的标签都需要直接写在类型或者包的上面，而有一些标签则需要用空行分隔一下。

**最佳实践就是根据一个模板来写**，例如：

- kubernets 官方的 [sample-controller](https://github.com/kubernetes/sample-controller)。
- knative 官方的 [sample-resource](https://github.com/knative-sandbox/sample-source)。

#### 全局标签

全局的标签都是写在 `doc.go` 文件中，常见的路径是 `pkg/apis/<apigroup>/<version>/doc.go`，内容如下：

```go
// +k8s:deepcopy-gen=package,register
// +groupName=example.com

// Package v1 is the v1 version of the API.
package v1
```

第一行注释中标签含义是：指示 `deepcopy-gen` 通过默认方式为这个包中的每一个类型都创建 `deepcopy()` 方法。如果某个类型不需要 `deepcopy()` 方法，那么在该类型上增加局部标签 `// +k8s:deepcopy-gen=false`。如果没有全局的 `deepcopy` 标签（包级别），那么需要为包中的每个类型都添加一个局部标签 `// +k8s:deepcopy-gen=true`。

> 注意：第一行注释中的 `register` 值会将整个包中的 `deepcoyp()` 方法都注册到 scheme 中。在1.9版本之后就没有这个关键字了，因为 scheme 不再负责 `runtime.Object` 任何深拷贝。取而代之的是使用 `yourobject.DeepCopy()` 或者 `yourobject.DeepCopyObject()` 。

第二行注释中标签含义是：定义标准 API 的组名（即 k8s 中的 group），这里如果写错了，那么 `client-gen` 就会生成错误的代码。**注意，这个标签必须在package上方的注释中**。

#### 局部标签

通常将 API 类型的源代码保存在 `xxx_types.go` 结尾的文件中。局部标签要么直接写在 API 类型上，要么写在 API 类型上的第二个注释块中。如下是一个 CR 的示例：

```go
// +genclient
// +genclient:noStatus
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// Database describes a database.
type Database struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec DatabaseSpec `json:"spec"`
}

// DatabaseSpec is the spec for a Foo resource
type DatabaseSpec struct {
    user string `json:"user"`
    Password string `json:"password"`
    Encoding string `json:"encoding,omitempty"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// DatabaseList is a list of Database resources
type DatabaseLists struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata"`

    Items []Database `json:"items"`
}
```

上面的注释中默认为所有的类型都启用了 `deepcopy`，因为这些类型都是 API 类型，所以不需要在这个源文件中对 `deepcopy` 进行设置，只需要在包级别的 `doc.go` 中设置全局标签即可。

`// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object` 是一个比较特殊的deepcopy标签。在1.8版本中 `runtime.Object` 接口使用了此方法签名进行扩展，因此每个 `runtime.Object` 都要实现`DeepCopyObject` 。具体的实现如下所示：

```go
func(in *T) DeepCopyObject() runtime.Object {
    if c := in.DeepCopy(); c != nil {
        return c
    } else {
        return nil
    }
}
```

不需要为每个类型都实现，只需要将 `// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object` 这个局部标签放在顶层 API 类型上就行。在上面的例子中，`Database` 和 `DatabaseList` 是顶层 API 类型，因为它们被当作 `runtime.Object` 使用。常见的顶层 API 类型就是那些嵌入 `metav1.TypeMeta` 的类型，这些也是客户端使用 `client-gen` 生成的类型。

> 注意: `// +k8s:deepcopy-gen:interfaces` 标签应该在定义一些具有接口类型的 API 时使用，例如 `field SomeInterface`，然后 `// +k8s:deepcopy-gen:interfaces=example.com/pkg/apis/examples.SpneInterface` 标签将会生成 `DeepCopySomeInterface() SomeInterface` 方法。这允许它以类型正确的方式对这些字段进行深度拷贝。

### client-gen 标签

有一些用于控制 `client-gen` 的标签，如下所示；

```go
// +genclient
// +genclient:noStatus
```

- 第一行注释中的标签：指示 `client-gen` 为这个 API 类型创建一个客户端，请注意， API 对象的 List 类型不需要这个标签。
- 第二行注释中的标签：指示 `client-gen` 不需要为这个类型生成符合 spec-status 规范的通过 `/status` 操作的子资源。那么该 API 类型的客户端中就没有 `UpdataStatus()` 方法（`client-gen` 看到标准的值为 `status` 就会直接生成这个方法）。

对应没有 Namespace 范围限定的资源，使用 `// +genclient:nonNamespaced`。

对于有些客户端，可能还需要更细粒度的 HTTP 方法，通过使用如下的一组标签值来指示 `client-gen` 生成对应代码：

```go
// +genclient:noVerbs
// +genclient:onlyVerbs=create,delete
// +genclient:skipVerbs=get,list,create,update,patch,delete,deleteCollection,watch
// +genclient:method=Create,verb=create,result=k8s.io/apimachinery/pkg/apis/meta/v1.Status
```

- 前三行注释中的标签是不言自明的。
- 第四行注释中的标签的含义是：这个 API 类型只有 create 方法，对它的 create 操作只会返回 `metav1.Status` 中的内容。这个对于 CRD 来说用处不大，对于自定义 API Server 就很有帮助。

## 生成 YAML 文件

使用 k8s-sigs 维护的生成工具 [controller-tools](https://github.com/kubernetes-sigs/controller-tools)。能够生成的 YAML 文件包括 CRD，RBAC等，通过命令行参数和标签来控制。

需要生成的YAML文件也是根据 `xxx_types.go` 源码和对应的注释标签生成的。常见的标签如下：

```go
// +kubebuilder:validation:Optional
// +kubebuilder:validation:MaxItems=2
// +kubebuilder:printcolumn:JSONPath=".status.replicas",name=Replicas,type=string
```

kubebuilder项目提供了2个make命令：

- make manifests 用来生成 Kubernetes 对象的 YAML 文件，像 `CustomResourceDefinitions`，`WebhookConfigurations` 和 `RBAC roles`。
- make generate 用来生成代码，像 `runtime.Object/DeepCopy implementations`。（与上面代码生成部分原理相同）

通用的形式是这样的：

```go
+path:to:marker=val
+path:to:marker:arg1=val,arg2=val2
+path:to:marker

+controllertools:generateHelp[:category=<string>]
```

- Empty：(`+kubebuilder:validation:Optional`)：空标记，就像命令行中的布尔标记位，仅仅是指定来开启某些行为。
- Anonymous： (`+kubebuilder:validation:MaxItems=2`)：匿名标记，使用单个值作为参数。
- Multi-option： (`+kubebuilder:printcolumn:JSONPath=".status.replicas",name=Replicas,type=string`)：多选项标记，使用一个或多个命名参数。第一个参数与名称之间用冒号隔开，而后面的参数使用逗号隔开。参数的顺序没有关系。有些参数是可选的。

标记的参数可以是字符，整数，布尔，切片，或者 map 类型。 字符，整数，和布尔都应该符合 Go 语法：

```go
// +kubebuilder:validation:ExclusiveMaximum=false
// +kubebuilder:validation:Format="date-time"
// +kubebuilder:validation:Maximum=42
```

切片可以用大括号和逗号分隔来指定：

```go
// +kubebuilder:webhooks:Enum={"crackers, Gromit, we forgot the crackers!","not even wensleydale?"}
```

Maps 是用字符类型的键和任意类型的值（有效地`map[string]interface{}`）来指定的。一个 map 是由大括号（`{}`）包围起来的，每一个键和每一个值是用冒号（`:`）隔开的，每一个键值对是由逗号隔开的。

```go
// +kubebuilder:validation:Default={magic: {numero: 42, stringified: forty-two}}
```

### CRD字段验证

CRD 支持使用 [OpenAPI v3 schema](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md#schemaObject) 在 validation 段中进行声明式验证。通常验证标记会关联到字段或者类型上。

如果定义了复杂的验证，或者如果需要重复使用验证，或者需要验证切片元素，那么最好定义一个新的类型来描述验证，如下下面例子中的 Alias 和 Rank 类型。

```go
type ToySpec struct {
    // +kubebuilder:validation:MaxLength=15
    // +kubebuilder:validation:MinLength=1
    Name string `json:"name,omitempty"`

    // +kubebuilder:validation:MaxItems=500
    // +kubebuilder:validation:MinItems=1
    // +kubebuilder:validation:UniqueItems=true
    Knights []string `json:"knights,omitempty"`

    Alias   Alias   `json:"alias,omitempty"`
    Rank    Rank    `json:"rank"`
}

// +kubebuilder:validation:Enum=Lion;Wolf;Dragon
type Alias string

// +kubebuilder:validation:Minimum=1
// +kubebuilder:validation:Maximum=3
// +kubebuilder:validation:ExclusiveMaximum=false
type Rank int32
```

### kubectl get 命令中显示其他列信息

从 Kubernetes 1.11 开始，`kubectl get` 可以询问 Kubernetes 服务要展示哪些列。对于 CRD 来说，可以用 `kubectl get` 来提供展示有用的特定类型的信息，类似于为内置类型提供的信息。

CRD 的 `additionalPrinterColumns` 和 `kube-additional-printer-columns` 字段控制了要展示的信息，它是通过在 Go 类型上标注 `+kubebuilder:printcolumn` 标签来控制要展示的信息。

```go
// +kubebuilder:printcolumn:name="Alias",type=string,JSONPath=`.spec.alias`
// +kubebuilder:printcolumn:name="Rank",type=integer,JSONPath=`.spec.rank`
// +kubebuilder:printcolumn:name="Bravely Run Away",type=boolean,JSONPath=`.spec.knights[?(@ == "Sir Robin")]`,description="when danger rears its ugly head, he bravely turned his tail and fled",priority=10
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"
type Toy struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   ToySpec   `json:"spec,omitempty"`
    Status ToyStatus `json:"status,omitempty"`
}
```

### 子资源

在 Kubernetes 1.13 中 CRD 可以选择实现 `/status` 和 `/scale`这类子资源。推荐在所有资源上都实现 `/status` 子资源，那么对应的API 类型上要有一个状态（status）字段。

#### 状态

通过 `+kubebuilder:subresource:status` 设置子资源的状态。启用状态时，更新主资源不会修改它的状态。类似的，更新子资源状态也只是修改了状态字段。

```go
// +kubebuilder:subresource:status
type Toy struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   ToySpec   `json:"spec,omitempty"`
    Status ToyStatus `json:"status,omitempty"`
}
```

#### 扩缩容

子资源的伸缩可以通过 `+kubebuilder:subresource:scale` 来启用。启用后，使用 `kubectl scale` 来对资源进行扩缩容。

如果 `selectorpath` 参数被指定为字符串形式的标签选择器，HorizontalPodAutoscaler 将可以自动扩容这些资源。

```go
type CustomSetSpec struct {
    Replicas *int32 `json:"replicas"`
}

type CustomSetStatus struct {
    Replicas int32 `json:"replicas"`
    Selector string `json:"selector"` // this must be the string form of the selector
}


// +kubebuilder:subresource:status
// +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.replicas,selectorpath=.status.selector
type CustomSet struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   ToySpec   `json:"spec,omitempty"`
    Status ToyStatus `json:"status,omitempty"`
}
```
