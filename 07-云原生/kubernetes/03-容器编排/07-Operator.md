# Kubernetes Operator

管理**有状态应用**的另一个解决方方案：Operator。

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

## 0.1. 例子

Etcd Operator。

1. 克隆仓库

  ```bash
  git clone https://github.com/coreos/etcd-operator

  ```

2. 部署Operator

```bash
$ example/rbac/create_role.sh
# 为Etcd Operator创建RBAC规则，
# 因为Etcd Operator需要访问APIServer
```

具体的为Etcd OPerator定义了如下所示的权限：

1. 具有Pod、Service、PVC、Deployment、Secret等API对象的所有权限
2. 具有CRD对象的所有权限
3. 具有属于etcd.database.coreos.com这个API Group的CR对象的所有权限

Etcd Operator本身是一个Deployment，如下所示：

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: etcd-operator
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: etcd-operator
    spec:
      containers:
      - name: etcd-operator
        image: quay.io/coreos/etcd-operator:v0.9.2
        command:
        - etcd-operator
        env:
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
...

```

创建这个Etcd Operator：

```bash
$ kubectl create -f example/deployment.yaml

$ kubectl get pods
NAME                              READY     STATUS      RESTARTS   AGE
etcd-operator-649dbdb5cb-bzfzp    1/1       Running     0          20s

$ kubectl get crd
NAME                                    CREATED AT
etcdclusters.etcd.database.coreos.com   2018-09-18T11:42:55Z
```

有一个名叫etcdclusters.etcd.database.coreos.com的CRD被创建，查看它的具体内容：

```bash
$ kubectl describe crd  etcdclusters.etcd.database.coreos.com
...
Group:   etcd.database.coreos.com
  Names:
    Kind:       EtcdCluster
    List Kind:  EtcdClusterList
    Plural:     etcdclusters
    Short Names:
      etcd
    Singular:  etcdcluster
  Scope:       Namespaced
  Version:     v1beta2
...
```

这个CRD告诉kubernetes集群，如果有API组（Group）是etcd.database.coreos.com，API资源类型（Kind）是EtcdCluster的YAML文件被提交时，就能够认识它。

> 上述操作是在集群中添加了一个名叫EtcdCluster的自定义资源类型，Etcd Operator本身就是这个自定义资源类型对应的自定义控制器。

Etcd Operator部署好之后，在集群中创建Etcd集群的工作就直接编写EtcdCluster的YAML文件就可以，如下：

```bash
$ kubectl apply -f example/example-etcd-cluster.yaml
# example-etcd-cluster.yaml文件描述了3个节点的Etcd集群

$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
example-etcd-cluster-dp8nqtjznc   1/1       Running     0          1m
example-etcd-cluster-mbzlg6sd56   1/1       Running     0          2m
example-etcd-cluster-v6v6s6stxd   1/1       Running     0          2m

```

具体看一下`example-etcd-cluster.yaml`的文件内容，如下：

```yaml
apiVersion: "etcd.database.coreos.com/v1beta2"
kind: "EtcdCluster"
metadata:
  name: "example-etcd-cluster"
spec:
  size: 3
  version: "3.2.13"

```

这个yaml文件的内容很简单，只有集群节点数3，etcd版本3.2.13，具体创建集群的逻辑有Etcd Operator完成。

## 0.2. Operator工作原理

1. 利用kubernetes的自定义API资源（CRD）来描述需要部署的**有状态应用**
2. 在自定义控制器里，根据自定义API对象的变化，来完成具体的部署和运维工作

> 编写Operator和编写自定义控制器的过程，没什么不同。

## 0.3. Etcd集群的构建方式

Etcd Operator部署Etcd集群，采用的是**静态集群**（Static）的方式。

静态集群：

- 好处：它不必依赖于一个额外的**服务发现机制**来组建集群，非常适合本地容器化部署。
- 难点：必须在部署的时候就规划好这个集群的拓扑结构，并且能够知道这些节点固定的IP地址，如下所示。

```bash
$ etcd --name infra0 --initial-advertise-peer-urls http://10.0.1.10:2380 \
  --listen-peer-urls http://10.0.1.10:2380 \
...
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new
  
$ etcd --name infra1 --initial-advertise-peer-urls http://10.0.1.11:2380 \
  --listen-peer-urls http://10.0.1.11:2380 \
...
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new
  
$ etcd --name infra2 --initial-advertise-peer-urls http://10.0.1.12:2380 \
  --listen-peer-urls http://10.0.1.12:2380 \
...
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new

```

启动三个Etcd进程，组建三节点集群。当infra2节点启动后，这个Etcd集群中就会有infra0、infra1、infra2三个节点。节点的启动参数`-initial-cluster`**正是当前节点启动时集群的拓扑结构，也就是当前界定在启动的时候，需要跟那些节点通信来组成集群**。

- `--initial-cluster`参数是由“<节点名字>=<节点地址>”格式组成的一个数组。
- `--listen-peer-urls`参数表示每个节点都通过2380端口进行通信，以便组成集群。
- `--initial-cluster-token`字段，表示集群独一无二的Token。

编写Operator就是要把上述对每个节点进行启动参数配置的过程自动化完成，即使用代码生成每个Etcd节点Pod的启动命令，然后把它们启动起来。

## 0.4. Etcd Operator构建过程

## 0.5. 编写EtcdCluster这个CRD

CRD对应的内容在`types.go`文件中，如下所示：

```go
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

type EtcdCluster struct {
  metav1.TypeMeta   `json:",inline"`
  metav1.ObjectMeta `json:"metadata,omitempty"`
  Spec              ClusterSpec   `json:"spec"`
  Status            ClusterStatus `json:"status"`
}

type ClusterSpec struct {
 // Size is the expected size of the etcd cluster.
 // The etcd-operator will eventually make the size of the running
 // cluster equal to the expected size.
 // The vaild range of the size is from 1 to 7.
 Size int `json:"size"`
 ...
}
```

EtcdCluster是一个有Status字段的CRD，在Spec中只需要关心Size（集群的大小）字段，这个字段意味着需要调整集群大小时，直接修改YAML文件即可，Operator会自动完成Etcd节点的增删操作。

**这种scale能力，也是Etcd Operator自动化运维Etcd集群需要实现的主要功能**。为了实现这个功能，不能在`--initial-cluster`参数中把拓扑结构固定死。所有Etcd Operator在构建集群时，虽然也是静态集群，但是是通过逐个节点动态添加的方式实现。

## 0.6. Operator创建集群

1. Operator创建“种子节点”
2. Operator创建新节点，逐一加入集群中，直到集群节点数等于size

生成不同的Etcd Pod时，Operator要能够区分种子节点和普通节点，这两个节点的不同之处在`--initial-cluster-state`这个启动参数：

- 参数值设为new，表示为种子节点，**种子节点不需要通过`--initial-cluster-token`声明独一无二的Token**
- 参数值为existing，表示为普通节点，Operator将它加入已有集群

> 需要注意，种子节点启动时，集群中只有一个节点，即`--initial-cluster`参数的值为`infra0=<http://10.0.1.10:2380>`，其他节点启动时，节点个数依次增加，即`--initial-cluster`参数的值不断变化。

### 0.6.1. 启动种子节点

用户提交YAML文件声明要创建EtcdCluster对象，Etcd Operator先创建一个单节点的种子集群，并启动它，启动参数如下：

```bash
$ etcd
  --data-dir=/var/etcd/data
  --name=infra0
  --initial-advertise-peer-urls=http://10.0.1.10:2380
  --listen-peer-urls=http://0.0.0.0:2380
  --listen-client-urls=http://0.0.0.0:2379
  --advertise-client-urls=http://10.0.1.10:2379
  --initial-cluster=infra0=http://10.0.1.10:2380    # 目前集群只有一个节点
  --initial-cluster-state=new       # 参数值为new表示是种子节点
  --initial-cluster-token=4b5215fa-5401-4a95-a8c6-892317c9bef8      # 种子节点需要唯一指定token
```

这个创建种子节点的阶段称为：Bootstrap。

### 0.6.2. 添加普通节点

对于其他每个节点，Operator只需要执行如下两个操作即可：

```bash
# 通过Etcd命令行添加新成员
$ etcdctl member add infra1 http://10.0.1.11:2380

# 为每个成员节点生成对应的启动参数，并启动它
$ etcd
    --data-dir=/var/etcd/data
    --name=infra1
    --initial-advertise-peer-urls=http://10.0.1.11:2380
    --listen-peer-urls=http://0.0.0.0:2380
    --listen-client-urls=http://0.0.0.0:2379
    --advertise-client-urls=http://10.0.1.11:2379
    --initial-cluster=infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380     #目前集群有两个节点
    --initial-cluster-state=existing        # 参数值为existing表示为普通节点，并且不需要唯一的token
```

继续添加，直到集群数量变成size为止。

## 0.7. Etcd Operator工作原理

与其他自定义控制器一样，Etcd Operator的启动流程也是围绕Informer，如下：

```go
func (c *Controller) Start() error {
 for {
  err := c.initResource()
  ...
  time.Sleep(initRetryWaitTime)
 }
 c.run()
}

func (c *Controller) run() {
 ...

 _, informer := cache.NewIndexerInformer(source, &api.EtcdCluster{}, 0, cache.ResourceEventHandlerFuncs{
  AddFunc:    c.onAddEtcdClus,
  UpdateFunc: c.onUpdateEtcdClus,
  DeleteFunc: c.onDeleteEtcdClus,
 }, cache.Indexers{})

 ctx := context.TODO()
 // use workqueue to avoid blocking
 informer.Run(ctx.Done())
}
```

Etcd Operator：

1. 第一步，创建EtcdCluster对象所需的CRD，即etcdclusters.etcd.database.coreos.com
2. 第二步，定义EtcdCluster对象的Informer

**注意，Etcd Operator并没有使用work queue来协调Informer和控制循环。**

> 因为在控制循环中执行的业务逻辑（如创建Etcd集群）往往比较耗时，而Informer的WATCH机制对API对象变化的响应，非常迅速。所以控制器里的业务逻辑会拖慢Informer的执行周期，甚至可能block它，要协调快慢任务典型的解决方案，就是引入工作队列。

在Etcd Operator里没有工作队列，在它的EventHandler部分，就不会有入队的操作，而是直接就是每种事件对应的具体的业务逻辑。Etcd Operator在业务逻辑的实现方式上，与常规自定义控制器略有不同，如下所示：

![image](https://static001.geekbang.org/resource/image/e7/36/e7f2905ae46e0ccd24db47c915382536.jpg)

> 不同之处在于，Etcd Operator为每一个EtcdCluster对象都启动一个控制循环，并发地响应这些对象的变化。**这样不仅可以简化Etcd Operator的代码实现，还有助于提高响应速度**。

## 0.8. Operator与StatefulSet对比

1. StatefulSet里，它为Pod创建的名字是带编号的，这样就把整个集群的拓扑状态固定，而在Operator中名字是随机的

  > Etcd Operator在每次添加节点或删除节点时都执行`etcdctl`命令，整个过程会更新Etcd内部维护的拓扑信息，所以不需要在集群外部通过编号来固定拓扑关系。

2. 在Operator中没有为EtcdCluster对象声明Persistent Volume，在节点宕机时，是否会导致数据丢失？

- Etcd是一个基于Raft协议实现的高可用键值对存储，根据Raft协议的设计原则，当Etcd集群里只有半数以下的节点失效时，当前集群依然可用，此时，Etcd Operator只需要通过控制循环创建出新的Pod，然后加入到现有集群中，就完成了期望状态和实际状态的调谐工作。
- 当集群中半数以上的节点失效时，这个集群就会丧失数据写入能力，从而进入“不可用”状态，此时，即使Etcd Operator 创建出新的Pod出来，Etcd集群本身也无法自动恢复起来。**这个时候就必须使用Etcd本身的备份数据（由单独的Etcd Backup Operator完成）来对集群进行恢复操作**。

创建和使用Etcd Backup Operator的过程：

```bash
# 首先，创建 etcd-backup-operator
$ kubectl create -f example/etcd-backup-operator/deployment.yaml

# 确认 etcd-backup-operator 已经在正常运行
$ kubectl get pod
NAME                                    READY     STATUS    RESTARTS   AGE
etcd-backup-operator-1102130733-hhgt7   1/1       Running   0          3s

# 可以看到，Backup Operator 会创建一个叫 etcdbackups 的 CRD
$ kubectl get crd
NAME                                    KIND
etcdbackups.etcd.database.coreos.com    CustomResourceDefinition.v1beta1.apiextensions.k8s.io

# 我们这里要使用 AWS S3 来存储备份，需要将 S3 的授权信息配置在文件里
$ cat $AWS_DIR/credentials
[default]
aws_access_key_id = XXX
aws_secret_access_key = XXX

$ cat $AWS_DIR/config
[default]
region = <region>

# 然后，将上述授权信息制作成一个 Secret
$ kubectl create secret generic aws --from-file=$AWS_DIR/credentials --from-file=$AWS_DIR/config

# 使用上述 S3 的访问信息，创建一个 EtcdBackup 对象
$ sed -e 's|<full-s3-path>|mybucket/etcd.backup|g' \
    -e 's|<aws-secret>|aws|g' \
    -e 's|<etcd-cluster-endpoints>|"http://example-etcd-cluster-client:2379"|g' \
    example/etcd-backup-operator/backup_cr.yaml \
    | kubectl create -f -
```

**注意，每次创建一个EtcdBackup对象，就相当于为它所指定的Etcd集群做了一次备份**。EtcdBackup对象的etcdEndpoints字段，会指定它要备份的Etcd集群的访问地址。在实际环境中，可以把备份操作编写成一个CronJob。

当Etcd集群发生故障时，可以通过创建一个EtcdRestore对象来完成恢复操作。需要事先创建Etcd Restore Operator，如下：

```bash
# 创建 etcd-restore-operator
$ kubectl create -f example/etcd-restore-operator/deployment.yaml

# 确认它已经正常运行
$ kubectl get pods
NAME                                     READY     STATUS    RESTARTS   AGE
etcd-restore-operator-4203122180-npn3g   1/1       Running   0          7s

# 创建一个 EtcdRestore 对象，来帮助 Etcd Operator 恢复数据，记得替换模板里的 S3 的访问信息
$ sed -e 's|<full-s3-path>|mybucket/etcd.backup|g' \
    -e 's|<aws-secret>|aws|g' \
    example/etcd-restore-operator/restore_cr.yaml \
    | kubectl create -f -
```

当一个EtcdRestore对象创建成功之后，Etcd Restore Operator就会通过上述信息，恢复出一个全新的Etcd集群，然后Etcd Operator会把这个新的集群直接接管从而重新进入可用状态。
