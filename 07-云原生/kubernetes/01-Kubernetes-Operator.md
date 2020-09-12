# Kubernetes Operator

## Operator

> Operator 是 Kubernetes 的重要扩展机制，旨在管理一个或一组服务的关键目标。负责特定应用和 Service 的 Operator，在系统应该如何运行、如何部署以及出现问题时如何处理等方面有深入的了解。

在 Kubernetes 上运行工作负载都喜欢通过自动化来处理重复的任务。Operator 模式会封装编写的（Kubernetes 本身提供功能以外的）任务自动化代码。

Operator 通过扩展 Kubernetes 控制平面和 API 进行工作。Operator 将一个 endpoint（称为自定义资源 CR）添加到 Kubernetes API 中，该 endpoint 还包含一个监控和维护新类型资源的控制平面组件。

> Operator与Controller：Operator 由一组监听 Kubernetes 资源的 Controller 组成。Controller 可以实现调协（reconciliation loop），另外每个 Controller 都负责监视一个特定资源，当创建、更新或删除受监视的资源时就会触发调协。

有一些用于创建 Kubernetes Operator 的开源项目，例如：

- [Operator SDK](https://github.com/operator-framework/operator-sdk)
- [Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)
- [KUDO](https://github.com/kudobuilder/kudo)
- [Metacontroller](https://github.com/GoogleCloudPlatform/metacontroller)【不维护了】

在 GitHub 上，有两个不同的开源项目用于创建 Operator，现在它们为实现同一目标而共同努力，就是Operator SDK 和 Kubebuilder，是最常用到的工具，这二者现在还在互相融合。它们之前生成代码，是不同的项目结构，但现在可以使用相同的结构样式。

## Kubernetes Controller

Controller 是一个非终止循环，用于调节系统状态，它会使 current 状态尽可能接近 desired 状态（亦称：调协，Reconciliation loop）。

![reconciliation loop](/images/controller-loop.png)

在Kubernetes中有一组内置的Controller在主节点的Controller-Manager内部运行：

- Deployment
- ReplicaSet
- DaemonSet
- StatefulSet
- Endpoint
- Service
- CronJon
- Job

与内置 Controller 类似，可以创建自己的自定义 Operator 来管理应用程序资源的状态，无论是无状态还是有状态 。

## 创建 Operator

使用 Operator SDK 项目。

### Operator SDK

Operator-SDK 是 Operator Framework 的组件，用于创建 Kubernetes 本机应用程序所需的代码。

![Operator SDK](/images/Operator-sdk.png)

Operator Framework 是一个开放源代码工具包，使用有效、自动化和可扩展的方式管理 Kubernetes 本地应用程序，包括：

- Operator SDK
- Operator Lifecycle Management（OLM）
- Operator Meterimg

Operator-SDK 允许创建三种不同类型的运算符：

- Helm：创建一个 Operator，使用 Helm Charts 管理创建的 Kubernetes 资源生命周期（CRUD）。
- Ansible：与 Helm 类似，创建 Operator 来管理 Ansible playbook 和 role，以对跟踪的 Kubernetes 资源（通常是 CR）更改做出反应。
- Go：与 Helm 和 Ansible 不同，基于 Golang 的 Operator 需要创建自定义逻辑，以监控资源以及协调应用程序状态。相对而言，它更为复杂，但它可以提供自由灵活的方式来实现想要的逻辑。

> Operator Libraries：Operator 利用 library 与 Kubernetes API 进行交互，例如 client-go 和 controller-runtime。了解它们的工作方式（Informer、Lister、WorkQueue、runtime.Object 和 Scheme）非常重要，如果创建 Go Operator，那就需要编写代码。

Operator 通常会对资源（Deployment、Job、Secret、Service、Pod 等）进行 CRUD 操作，并更新它们的状态。利用 go 模板或第三方库（例如 Manisfestival）可以使用程序模板或声明性方法来创建或编辑资源。

GitHub 上有一个不错的精选列表，叫 [Awesome Operators](github.com/operator-framework/awesome-operators)，它有很多 Operator 脚手架工具（scaffolding tool）创建的不同项目。

## Operator Hub

Kubernetes 社区创建了一个 Operator 托管场所，称为 Operator Hub。在这里，我们可以发布我们的 Operator，类似于其他 Hub，例如 Docker、Helm 等。

### Helm vs Operator

如果使用 Operator 管理 Kubernetes 生命周期资源（例如 CRUD 操作），为什么不用 Helm？

Helm 是针对第 1 天的操作，而 Operator 则针对第 2 天的操作。

- 第 0 天：软件开发中，代表了设计阶段，在此阶段收集解决方案的所有要求。
- 第 1 天：首次安装应用程序和基础架构的时间。
- 第 2 天：管理生产中应用程序和软件的生命周期，以确保一切都正常运行，如备份、还原、故障转移、后备。

> 例如，通过 Helm Charts 安装了一个应用程序（假设它创建了 Deployment、Service 和 Ingress），然后不小心删除了 Service，应用程序将停止运行。从 Helm 角度来看，在应用新配置之前，它看上去是正常的，我们不会意识到更改。这就是 Operator 发挥作用的地方，在这个例子中，如果有人误删除了 Service，并且 Operator 正在监控该资源，它将在恢复过程中重新创建，因此应用程序将恢复正常。Helm 和 Operators 是互补的，不是互斥的。
