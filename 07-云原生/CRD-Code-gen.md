# Code Gen for CRD

## 简介

client-go要求`runtime.Object`类型必须有一个`DeepCopy`方法。也就是说，Go语言实现的CR必须实现`runtime.Object`接口。

代码生成器，通过deepcopy-gen来生成对应的代码。deepcode-gen主要提供如下功能：

- deepcopy-gen：为每一种类型提供一个DeepCopy方法，`func(t* T) DeepCopy() *T`
- client-gen：为CR的API组创建类型化的客户端集合
- informer-gen：为CR创建informer，用途是基于事件接口，对服务器上CR的变更作出响应
- lister-gen：为CR创建lister，用途是对GET和LIST请求提供一个只读的缓存层

> 后面两个是控制器的基础。这四个代码生成器是构建功能齐全，生产可用的控制器的基础。和Kubernetes内置控制器使用的机制和库都是一样的。更多其他的生成器点[这里](https://github.com/kubernetes/code-generator)。例如在创建聚合API服务器的时候，conversion-gen就用于实现内外部类型之间的转换功能。defaulter-gen则用于一些默认的字段。

## 脚本使用

所有的Kubernetes代码生成器都是在[gengo](https://github.com/kubernetes/gengo)的基础上创建的，它们共享很多公用的命令行标识。通常，所有的生成器都是从`--input-dirs`中获取输入，然后一个类型一个类型的转换为生成的代码。

- 生成的代码与输入代码在同一个路径下，像deepcopy-gen那样通过`--output-file-base "zz_generated.deepcopy"`来定义输出的文件名。
- 或者生成的代码输出在一个或多个包中，使用`--output-package`来指定，输出类似`client-`，`informer-`，就像lister-gen那样通常输出在`pkg/client`。

代码生成器提供来一个[shell脚本](https://github.com/kubernetes/code-generator/blob/master/generate-groups.sh)，这样就不需要熟悉那么多的命令行参数来。通常脚本保存在`hack/update-codegen.sh`。生成后，所有的API都在`pkg/apis`，所有的clientsets、informers、listers都在`pkg/client`中。然后是`hack/verify-codegen.sh`脚本文件，如果生成的文件不是最新的，那么执行后会返回一个非零的输出，这在CI中很有用。

## 代码生成标签

代码生成器的一些行为可以通过命令行参数来控制，但是更多的属性可以通过在go源文件中的标签来控制。有两种类型的标签：

- 全局标签，作用在package上，通常在doc.go文件中
- 局部标签，作用在type上

通常标签看起来是这样的：`// +tag-name`或者`// +tag-name=value`，都是写在注释中的，因此注释的位置就显得很重要了。大部分的标签都需要直接写在类型或者包上面，而有一些标签则需要用空行分隔一下。最佳实践就是根据一个模板来写。

### 全局标签

全局的标签都是写在`doc.go`文件中的，通常在这个路径下，`pkg/apis/<apigroup>/<version>/doc.go`，内容如下：

```go
// +k8s:deepcopy-gen=package,register
// Package v1 is the v1 version of the API.
// +groupName=example.com
package v1
```

上面的标签告诉deepcopy-gen通过默认方式为这个包中的每一个类型都创建deepcopy方法。如果某个类型不需要deepcopy方法，那么在那个类型上增加局部标签`// +k8s:deepcopy-gen=false`。如果没有开启包级别的deepcopy，那么需要为包中的每个类型都添加一个标签`// +k8s:deepcopy-gen=true`。

> 注意：上面例子中的register噶u你就爱你这会将整个包中的deepcoyp方法都注册到scheme中。在1.9版本之后就没有这个关键字了，因为scheme不再负责`runtime.Object`任何深拷贝。取而代之的是使用`yourobject.DeepCopy()`或者`yourobject.DeepCopyObject()`。

`// +groupName=example.com`定义标准API的组名，这里如果写错了，那么client-gen就会生成错误的代码。**注意，这个标签必须在package上方的注释中**。

### 局部标签

局部标签要么直接写在API类型上，要么写在API类型上的第二个注释块中。如下是一个CR的示例：

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

上面的注释中默认为所有的类型都启用了deepcopy，因为这些类型都是API类型，所有不需要在这个源文件中对deepcopy进行设置，只需要在包级别的doc.go中设置全局标签即可。

### runtime.Object和DeepCopyObject

`// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object`这是一个比较特殊的deepcopy标签。在1.8版本中`runtime.Object`接口使用了此方法签名进行扩展，因此每个`runtime.Object`都要实现DeepCopyObject。具体的实现如下所示：

```go
func(in *T) DeepCopyObject() runtime.Object {
    if c := in.DeepCopy(); c != nil {
        return c
    } else {
        return nil
    }
}
```

不需要为每个类型都实现，只需要将`// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object`这个局部标签放在顶层API类型上就行。在上面的例子中，Database和DatabaseList是顶层API类型，因为它们被当作`runtime.Object`使用。约定俗称的顶层API是那些嵌入`metav1.TypeMeta`的类型，同样这些也是客户端使用client-gen生成的类型。

注意`// +k8s:deepcopy-gen:interfaces`标签应该在定义一些具有接口类型的API时使用，例如`field SomeInterface`，然后`// +k8s:deepcopy-gen:interfaces=example.com/pkg/apis/examples.SpneInterface`标签将会导致生成`DeepCopySomeInterface() SomeInterface`方法。这允许它以类型正确的方式对这些字段进行深度拷贝。

### Client-gen 标签

有一些用于控制Client-gen的标签，如下所示；

```go
// +genclient

// +genclient:noStatus
```

上例中第一个标签告诉client-gen为这个类型创建一个客户端，请注意，不需要把这个标签在API对象的List类型上方。第二个标签告诉client-gen这个类型没有通过`/status`子资源使用spec-status规范。结果就是客户端中没有UpdataStatus方法（因为client-gen看到status字段就会直接生成这个方法）。

对于范围的资源，使用`// +genclient:nonNamespaced`。对于有些客户端，可能还需要控制更细粒度的HTTP方法，这可以通过使用一组标签来实现，如下所示：

```go
// +genclient:noVerbs

// +genclient:onlyVerbs=create,delete

// +genclient:skipVerbs=get,list,create,update,patch,delete,deleteCollection,watch

// +genclient:method=Create,verb=create,result=k8s.io/apimachinery/pkg/apis/meta/v1.Status
```

前三行的标签不言自明，第四行的意思是这个类型只会被创建并且不会返回这个类型自身，除了`metav1.Status`。这个对于CRD来说用处不大，对于自定义API Server就很有帮助。
