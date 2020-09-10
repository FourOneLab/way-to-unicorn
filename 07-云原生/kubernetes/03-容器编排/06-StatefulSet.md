---
title: 06-StatefulSet
date: 2020-04-14T10:09:14.162627+08:00
draft: false
---

- [0.1. 拓扑状态](#01-拓扑状态)
  - [0.1.1. Headless Service](#011-headless-service)
  - [0.1.2. 使用DNS记录来维持Pod的拓扑状态](#012-使用dns记录来维持pod的拓扑状态)
- [0.2. 存储状态](#02-存储状态)
- [0.3. StatefulSet工作原理](#03-statefulset工作原理)
  - [0.3.1. 总结](#031-总结)
  - [0.3.2. 滚动更新](#032-滚动更新)
- [0.4. 实践](#04-实践)
  - [0.4.1. 搭建MySQL集群](#041-搭建mysql集群)
    - [0.4.1.1. 第一步：备份主节点](#0411-第一步备份主节点)
    - [0.4.1.2. 第二步：配置从节点](#0412-第二步配置从节点)
    - [0.4.1.3. 第三步：启动从节点](#0413-第三步启动从节点)
    - [0.4.1.4. 第四步：添加从节点](#0414-第四步添加从节点)
  - [0.4.2. 迁移到kubernetes集群中](#042-迁移到kubernetes集群中)
    - [0.4.2.1. 问题一：主从节点需要不同的配置文件](#0421-问题一主从节点需要不同的配置文件)
    - [0.4.2.2. 问题二：主从节点需要传输备份文件](#0422-问题二主从节点需要传输备份文件)
      - [0.4.2.2.1. 第一步，从ConfigMap中，获取MySQL的Pod对应的配置文件](#04221-第一步从configmap中获取mysql的pod对应的配置文件)
      - [0.4.2.2.2. 第二步：在Slave Pod启动前，从Master或其他Slave里拷贝数据库数据到自己的目录下](#04222-第二步在slave-pod启动前从master或其他slave里拷贝数据库数据到自己的目录下)
      - [0.4.2.2.3. 第三步，Slave角色的MySQL容器启动前，执行初始化SQL语句](#04223-第三步slave角色的mysql容器启动前执行初始化sql语句)
        - [0.4.2.2.3.1. 工作一：MySQL节点初始化](#042231-工作一mysql节点初始化)
        - [0.4.2.2.3.2. 工作二：启动数据传输服务](#042232-工作二启动数据传输服务)
  - [0.4.3. 创建PV](#043-创建pv)
  - [0.4.4. 小结](#044-小结)

Deployment并不足以覆盖所有的应用编排问题，因为它对应用做了一个简单的假设：
> 一个应用的所有Pod是完全一样的，他们互相之间没有顺序也无所谓运行在哪台宿主机上。需要的时候Deployment通过Pod模板创建新的Pod，不需要的时候，就可以“杀掉”任意一个Pod。

1. 在分布式应用中，多个实例之间并不是这样的关系，有很多的**依赖关系**（主从关系、主备关系）。
2. 数据存储类应用，它的多个实例往往都会在本地磁盘上保存一份数据。这些实例一旦被“杀掉”，即便重建出来，实例与数据之间的对应关系也丢失了，从而导致应用失败。

有状态应用：

- 实例之间有不对等关系
- 实例对外部数据有依赖关系

> 容器技术用于封装“无状态应用”尤其是Web服务，非常好，但是“有状态应用”就很困难。

kubernetes得益于“控制器模式”，在Deployment的基础上扩展出StatefulSet，它将应用抽象为两种情况：

1. 拓扑状态：应用的多个实例之间不是完全对等的关系。这些应用实例必须按照某些顺序启动。
2. 存储状态：应用的多个实例分别绑定了不同的存储数据。

> 比如应用的主节点A要先于从节点B启动，如果把A和B两个Pod删掉，它们被再次创建出来时，必须**严格按照这个顺序才行**，并且新建的Pod必须与原来的Pod的**网络标识一样**，这样原先的访问者才能使用同样的方法访问到这个新的Pod。
>
> 比如Pod A第一次读取到的数据应该和十分钟之后读取到的是同一份数据，哪怕在这期间Pod A被重新创建过，典型的例子就是一个数据库应用的多个存储实例。

**StatefulSet的核心功能，通过某种方式记录这些状态，然后在Pod被创建时，能够为新的Pod恢复这些状态**。

## 0.1. 拓扑状态

### 0.1.1. Headless Service

通过Service，可以访问对应的Deployment所包含的Pod。那么Service是如何被访问的：

1. 以Service的VIP（Virtual IP）方式：访问Service的VIP时，会把请求转发到该Servcice所代理的某一个Pod上。
2. 以Service 的DNS方式：比如通过my-svc.my-namespace.svc.cluster.local这条DNS可以访问到名为my-svc的Service所代理的某个Pod。
  **通过DNS具体可以分为两种方式**：
    1. Normal Service，访问my-svc.my-namespace.svc.cluster.local，解析到my-svc这个Service的VIP，然后与访问VIP的方式一样。
    2. Headless Service，访问my-svc.my-namespace.svc.cluster.local，解析到的直接就是my-svc代理的某个pod的IP地址。

**区别在于，Headless Servcice不需要分配VIP，可以直接以DNS记录的方式解析出被代理Pod的IP地址**。

```yaml
# headless service example
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None #这里是重点
  selector:
    app: nginx
```

Headless Service仍然是一个标准的Service的YAML文件，**只不过clusterIP字段为None**。这样的话，这个Service没有VIP作为头，被创建后不会被分配VIP，而是以DNS记录的方式暴露出它所代理的Pod。

1. 通过Label Selector筛选出需要被代理的Pod
2. 以上述方式创建的Headless Service之后，它所代理的Pod的IP地址，会被绑一个如下格式的DNS记录：

```bash
# 这个DNS是kubernetes为Pod分配的唯一的“可解析身份”
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
```

有了可解析身份，只要知道Pod的名字和对应的Service名字，就可以通过DNS记录访问到Pod的IP地址。

### 0.1.2. 使用DNS记录来维持Pod的拓扑状态

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
```

这个StatefulSet的YAML文件与同类型的Deployment的YAML文件的唯一区别是**多了一个`serviceName=nginx`字段**。

> 这个字段的作用，告诉StatefulSet控制器，在执行控制循环（control loop）的时候，使用nginx这个Headless Service来保证Pod的“可解析身份”。

此时执行创建任务，分别创建service和对应的StatefulSet：

```bash
$ kubectl create -f svc.yaml
$ kubectl get service nginx
NAME      TYPE         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx     ClusterIP    None         <none>        80/TCP    10s

$ kubectl create -f statefulset.yaml
$ kubectl get statefulset web
NAME      DESIRED   CURRENT   AGE
web       2         1         19s
```

查看StatefulSet的创建事件，或者使用kubectl的-w参数查看StatefulSet对应的Pod的创建过程：

```bash
$ kubectl get pods -w -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     0/1       Pending   0          0s
web-0     0/1       Pending   0         0s
web-0     0/1       ContainerCreating   0         0s
web-0     1/1       Running   0         19s
web-1     0/1       Pending   0         0s
web-1     0/1       Pending   0         0s
web-1     0/1       ContainerCreating   0         0s
web-1     1/1       Running   0         20s
```

StatefulSet给它所管理的Pod的名字进行了编号，从0开始，短横（-）相接，每个Pod实例一个，**绝不重复**。

> Pod的创建也按照编号顺序进行，只有当编号为0的Pod进入Running状态，并且细分状态为Ready之前，编号为1的pod都会一直处于pending状态。

**为Pod设置livenessProbe和readinessProbe很重要**。

当两个Pod都进入Running状态后，可以查看他们各自唯一的“网络身份”。

```bash
kubectl exec web-0 -- sh -c 'hostname'
web-0   //pod的名字与hostname一致
kubectl exec web-1 -- sh -c 'hostname'
web-1
```

以DNS的方式访问Headless Service，在启动的Pod的容器中，使用nslookup命令来解析Pod对应的Headlesss Service。

```bash
kubectl run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh
nslookup web-0.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.1.7

nslookup web-1.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.2.7
```

从nslookup命令的输出结果中发现，在访问`web-0.nginx`的时候，最后解析到的正是web-0这个pod的IP地址。

当删除这两个Pod后，会按照原先编号的顺序重新创建两个新的Pod，并且依然会分配与原来相同的“网络身份”。

**通过这种严格的对应规则，StatefulSet就保证了Pod网络标识的稳定性**。

> 通过这种方法，Kubernetes就成功地将Pod的拓扑状态（比如：哪个节点先启动，哪个节点后启动），按照“Pod名字+编号”的方式固定下来。并且Kubernetes还为每一个Pod提供了一个固定并且唯一的访问入口，即：**这个Pod对应的DNS记录**。

这些状态，在StatefulSet的整个生命周期里都保持不变，绝不会因为对应Pod的删除或重新创建而失效。

虽然`web-0.nginx`这条记录本身不会变化，但是它解析到的Pod的IP地址，并不是固定的，所以对于“有状态应用”实例的访问，必须使用DNS记录或者hostname的方式，绝不应该直接访问这些Pod的IP地址。

StatefulSet其实是Deployment的改良。通过Headless Service的方式，StatefulSet为每个Pod创建了一个固定并且稳定的DNS记录，来作为它的访问入口。

## 0.2. 存储状态

StatefulSet对存储状态的管理机制，主要是使用`Persistent Volume Claim`的功能。

> 在Pod的定义中可以声明Voluem（`spec.volumes`字段）,在这个字段里定义一个具体类型的Volume，如hostPath。

**当我们并不知道有哪些Volume类型（比如Ceph、GlusterFS）可用时，怎么办呢**？

```yaml
# Ceph RBD volume example
apiVersion: v1
kind: Pod
metadata:
  name: rbd
spec:
  containers:
    - image: kubernetes/pause
      name: rbd-rw
      volumeMounts:
      - name: rbdpd
        mountPath: /mnt/rbd
  volumes:
    - name: rbdpd
      rbd:
        monitors:
        - '10.16.154.78:6789'
        - '10.16.154.82:6789'
        - '10.16.154.83:6789'
        pool: kube
        image: foo
        fsType: ext4
        readOnly: true
        user: admin
        keyring: /etc/ceph/keyring
        imageformat: "2"
        imagefeatures: "layering"
```

1. 如果不懂Ceph RBD的使用方法，这个Pod的Volume字段基本看不懂。
2. 这个Ceph RBD对应的存储服务器、用户名、授权文件的位置都暴露出来了（信息被过度暴露）。

**Kubernetes引入了一组叫作PVC和PV的API对象，大大降低了用户声明和使用Volume的门槛**。

使用PVC来定义Volume，只要两步。
第一步： 定义一个PVC，声明想要的Volume属性。

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

不需要任何Volume细节的字段，只有描述的属性和定义。

- storage：1Gi【表示需要的Volume大小至少为1GiB】
- accessMode：ReadWriteOnce【表示Volume的挂载方式为可读写，并且只能被挂载到一个节点上，而不是多个节点共享】

volume类型和支持的访问模式，如下表。

|Volume Plugin|ReadWriteOnce|ReadOnlyMany|ReadWriteMany|
|---|---|---|---|
|AWSElasticBlockStore|✓|-|-
|AzureFile|✓|✓|✓
|AzureDisk|✓|-|-
|CephFS|✓|✓|✓
|Cinder|✓|-|-
|FC|✓|✓|-
|Flexvolume|✓|✓|depends on the driver
|Flocker|✓|-|-
|GCEPersistentDisk|✓|✓|-
|Glusterfs|✓|✓|✓
|HostPath|✓|-|-
|iSCSI|✓|✓|-
|Quobyte|✓|✓|✓
|NFS|✓|✓|✓
|RBD|✓|✓|-
|VsphereVolume|✓|-|- (works when pods are collocated)
|PortworxVolume|✓|-|✓
|ScaleIO|✓|✓|-
|StorageOS|✓|-|-

第二步：在Pod中声明使用这个PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
    - name: pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-storage
  volumes:
    - name: pv-storage
      persistentVolumeClaim:
        claimName: pv-claim
```

在这个pod的Volume定义中只需要声明它的类型是`persistentVolumeClaim`，然后指定PVC的名字，**完全不必关心Volume本身的定义**。

- 当我们创建这个Pod时，kubernetes会自动绑定一个符合条件的Volume。
- 这个Volume来自预先创建的PV（Persistent Volume）对象。

常见的PV对象如下：

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-volume
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  rbd:
    monitors:
    - '10.16.154.78:6789'
    - '10.16.154.82:6789'
    - '10.16.154.83:6789'
    pool: kube
    image: foo
    fsType: ext4
    readOnly: true
    user: admin
    keyring: /etc/ceph/keyring
    imageformat: "2"
    imagefeatures: "layering"
```

这个PV对象的`spec.rbd`字段，正是前面介绍的Ceph RBD Volume的详细定义。它声明的容量是10GiB，kubernetes会为刚才创建的PVC绑定这个PV。

> kubernetes中PVC和PV的设计，实际上**类似于“接口”和“实现”的思想**。这种解耦合，避免了因为向开发者暴露过多的存储系统细节而带来隐患。

- 开发者只需要知道并使用“接口”，即PVC；
- 运维人员负责给这个“接口”绑定具体的实现，即PV。

**PV和PVC的设计，使得StatefulSet对存储状态的管理成为了可能**。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```

为这个StatefulSet添加一个`volumeClaimTemplates`字段（类似于Deployment中PodTemplate的作用）。

凡是被这个StatefulSet管理的pod。都会声明一个对应的PVC，这个PVC的定义来自于`volumeClaimTemplates`这个模板字段。

**更重要的是，这个PVC的名字会被分配一个与这个Pod完全一致的编号**。

这个自动创建的PVC，与PV绑定成功后，就进入bound状态，这就意味着这个Pod可以挂载并使用这个PV。

**PVC是一种特殊的Volume**。一个PVC具体是什么类型的Volume，要在跟某个PV绑定之后才知道。PVC与PV能够绑定的前提是，在kubernetes系统中已经创建好符合条件的PV，或者在公有云上通过`Dynamic Provisioning`的方式，自动为创建的PVC匹配PV。

创建上述StatefulSet后，在集群中会出现两个PVC：

```bash
kubectl create -f statefulset.yaml
kubectl get pvc -l app=nginx
NAME        STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
www-web-0   Bound     pvc-15c268c7-b507-11e6-932f-42010a800002   1Gi        RWO           48s
www-web-1   Bound     pvc-15c79307-b507-11e6-932f-42010a800002   1Gi        RWO           48s
```

这些PVC都是以`<PVC名字>-<StatefulSet名字>-<编号>`的方式命名，并且处于Bound状态。

> 这个StatefulSet创建出来的Pod都会声明使用编号的PVC，比如名叫`web-0`的Pod的Volume字段就会声明使用`www-web-0`的PVC，从而挂载到这个PVC所绑定的PV。

当容器向这个Volume挂载的目录写数据时，都是写入到这个PVC所绑定的PV中。当这两个Pod被删除后，这两个Pod会被按照编号的顺序重新创建出来，原先与相同编号的Pod绑定的PV在Pod被重新创建后依然绑定在一起。

StatefulSet控制器恢复Pod的过程：

1. 当Pod被删除时，对应的PVC和PV并不会被删除，所以这个Volume里已经写入的数据，也依然会保存在远程存储服务里。
2. StatefulSet控制器发现，有Pod消失时，就会重新创建一个新的、名字相同的Pod来纠正这种不一致的情况。
3. 在这个新的Pod对象的定义里，它声明使用的PVC与原来的名字相同；这个PVC的定义来自PVC模板,这是StatefulSet创建Pod的标准流程。
4. 所有在这个新的Pod被创建出来后，kubernetes为它查找原来名字的PVC，就会直接找到旧的Pod遗留下来的同名的PVC，进而找到与这个PVC绑定在一起的PV。

这样新的Pod就可以挂载到旧Pod对应的那个Volume，并且获得到保存在Volume中的数据。

**通过这种方式，kubernetes的StatefulSet就实现了对应用存储状态的管理**。

## 0.3. StatefulSet工作原理

1. StatefulSet控制器直接管理Pod，因为StatefulSet里面不同的Pod实例，不再像ReplicaSet中那样都是完全一样的，而是有细微区别的。比如每个Pod的hostname、名字等都是不同的、都携带编号。

2. Kubernetes通过Headless Service，为这些有编号的Pod，在DNS服务器中生成带有同样编号的DNS记录。只要StatefulSet能够保证这些Pod名字里的编号不变，那么Service里类似于`<pod名字>.<svc名字>.<命名空间>.cluster.local`这样的DNS记录也就不会变，而这条记录解析出来的Pod的IP地址，则会随着后端Pod的删除和再创建而自动更新。**这是Service机制本身的能力，不需要StatefulSet操心**。

3. StatefulSet还为每一个Pod分配并创建一个同样编号的PVC。这样Kubernetes就可以通过Persistent Volume机制为这个PVC绑定上对应的PV，从而保证每个Pod都拥有独立的Volume。在这种情况下，即使Pod被删除，它所对应的PVC和PV依然会保留下来，所以当这个Pod被重新创建出来之后，Kubernetes会为它找到同样编号的PVC，挂载这个PVC对应的Volume，从而获取到以前保存在Volume里的数据。

### 0.3.1. 总结

StatefulSet其实就是一种特殊的Deployment，其独特之处在于，它的每个Pod都被编号。而且，这个编号会体现在Pod的名字和hostname等标识信息上，这不仅代表了Pod的创建顺序，也是Pod的重要网络标识（即：在整个集群里唯一的、可被访问的身份）。

有了这个编号后，StatefulSet就使用kubernetes里的两个标准功能：Headless Service和PV/PVC，实现了对Pod的拓扑状态和存储状态的维护。**StatefulSet是kubernetes中作业编排的`集大成者`**。

### 0.3.2. 滚动更新

StatefulSet编排“有状态应用”的过程，其实就是对现有典型运维业务的容器化抽象。也就是说，在不使用kubernetes和容器的情况下，也可以实现，只是在升级、版本管理等工程的能力很差。

> 使用StatefulSet进行“滚动更新”，只需要修改StatefulSet的Pod模板，就会自动触发“滚动更新”的操作。

```bash
kubectl patch statefulset mysql --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"mysql:5.7.23"}]' statefulset.apps/mysql patched
```

使用kubectl path命令，以“补丁”的方式（JSON格式的）修改一个API对象的指定字段，即`spec/template/spec/containers/0/image`。

**这样，StatefulSet Controller就会按照与Pod编号`相反`的顺序，从最后一个Pod开始，逐一更新这个StatefulSet管理的每个Pod**。如果发生错误，这次滚动更新会停止。

> StatefulSet的滚动更新允许进行更精细的控制如（金丝雀发布，灰度发布），即**应用的多个实例中，被指定的一部分不会被更新到最新的版本**。StatefulSet的`spec.updateStragegy.rollingUpdate`的`partition`字段。

如下命令，将StatefulSet的partition字段设置为2：

```bash
kubectl patch statefulset mysql \
  -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}' statefulset.apps/mysql patched
```

上面的操作等同于使用`kubectl edit`命令直接打开这个对象，然后把partition字段修改为2。这样当模板发生变化时，只有序号大于等于2的Pod会被更新到这个版本，并且如果删除或者重启序号小于2的Pod，它再次启动后，还是使用原来的模板。

## 0.4. 实践

### 0.4.1. 搭建MySQL集群

相比于Etcd、Cassandra等“原生”就考虑分布式需求的项目，MySQL以及很多其他的数据库项目，在分布式集群上搭建并不友好，甚至有点“原始”。**使用StatefulSet将MySQL集群搭建过程“容器化”**，部署过程如下：

1. 部署一个“主从复制（Master-Slave Replication）”的MySQL集群
2. 部署一个主节点（Master）
3. 部署多个从节点（Slave）
4. 从节点需要水平扩展
5. 所有的写操作只在主节点上执行
6. 读操作可以在所有节点上执行

典型的主从模式MySQL集群如下所示：

![image](https://static001.geekbang.org/resource/image/bb/02/bb2d7f03443392ca40ecde6b1a91c002.png)

> 在常规环境中，部署这样一个主从模式的MySQL集群的主要**难点**在于：如何让从节点能拥有主节点的数据，即：如何配置主（Master）从（Slave）节点的复制与同步。

#### 0.4.1.1. 第一步：备份主节点

所以在安装好MySQL的Master节点后，需要做的第一步工作：**通过XtraBackup将Master节点的数据备份到指定目录**。

> XtraBackup是业界主要使用的开源MySQL备份和恢复工具。

这个过程会自动在目标目录生成一个备份信息文件，名叫：xtrabackup_binlog_info。这个文件一般会包含如下两个信息：

```bash
cat xtrabackup_binlog_info
TheMaster-bin.000001     481
```

这两个信息会在接来配置Slave节点的时候用到。

#### 0.4.1.2. 第二步：配置从节点

Slave节点在第一次启动之前，需要先把Master节点的备份数据，连同备份信息文件，一起拷贝到自己的数据目录（`/var/lib/mysql`）下，然后执行如下SQL语句：

```sql
TheSlave|mysql> CHANGE MASTER TO
                MASTER_HOST='$masterip',
                MASTER_USER='xxx',
                MASTER_PASSWORD='xxx',
                MASTER_LOG_FILE='TheMaster-bin.000001',
                MASTER_LOG_POS=481;
```

其中，`MASTER_LOG_FILE`和`MASTER_LOG_POS`，就是上一步中备份对应的二进制日志（Binary Log）文件的名称和开始的位置（偏移量），也正是`xtrabackup_binlog_info`文件里的那两部分内容（即`TheMaster-bin.000001`和`481`）.

#### 0.4.1.3. 第三步：启动从节点

执行如下SQL语句来启动从节点：

```sql
TheSlave|mysql> START SLAVE;
```

Slave节点启动并且会使用备份信息文件中的二进制日志文件和偏移量，与主节点进行数据同步。

#### 0.4.1.4. 第四步：添加从节点

**注意：新添加的Slave节点的备份数据，来自于已经存在的Slave节点**。

所以，在这一步，需要将Slave节点的数据备份在指定目录。而这个备份操作会自动生成另一份备份信息文件，名叫：xtrabackup_slave_info。这个文件也包含`MASTER_LOG_FILE`和`MASTER_LOG_POS`字段。

然后再执行第二步和第三步。

### 0.4.2. 迁移到kubernetes集群中

从上述步骤不难免看出，将部署MySQL集群的流程迁移到kubernetes项目上，需要能够“容器化”地解决下面的“三个问题”。

1. Master与Slave需要有不同的配置文件（my.cnf）
2. Master与Slave需要能够传输备份信息文件
3. 在Slave第一次启动之前，需要执行一些初始化SQL操作

**由于MySQL本身同时拥有拓扑状态（主从）和存储状态（MySQL数据保存在本地）**，所以使用StatefulSet来部署MySQL集群。

#### 0.4.2.1. 问题一：主从节点需要不同的配置文件

为主从节点分别准备两份配置文件，然后根据pod的序号挂载进去。配置信息应该保存在ConfigMap里供Pod使用：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels:
    app: mysql
data:
  master.cnf: |
    # 主节点 MySQL 的配置文件
    [mysqld]
    log-bin
  slave.cnf: |
    # 从节点 MySQL 的配置文件
    [mysqld]
    super-read-only
```

定义`master.cnf`和`slave.cnf`两个MySQL配置文件。

- master.cnf：开启log-bin，即使用二进制文件的方式进行主从复制
- slave.cnf：开启super-read-only，即从节点会拒绝除了主节点的数据同步操作之外的所有写操作（对用户只读）。

在ConfigMap定义里的data部分，是key-value格式的。比如master.cnf就是这份配置数据的Key，而`“|”`后面的内容，就是这份配置数据的Value。这份数据将来挂载到Master节点对应的Pod后，就会在Volume目录里生成一个叫做master.cnf的文件。

然后创建两个Service来供StatefulSet以及用户使用，定义如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
```

- 相同点：这两个Service都代理了所有携带app=mysql标签的pod。端口映射都是用service的3306端口对应Pod的3306端口。
- 不同点：第一个service是headless service（`ClusterIP=None`），它的作用是通过为pod分配DNS记录来固定它的拓扑状态。比如`mysql-0.mysql`和`mysql-1.mysql`这样的DNS名字，其中编号为0的节点就是主节点。第二个Service是一个常规的Service。

规定：

1. 所有的用户请求都必须访问第二个Service被自动分配的DNS记录，即`mysql-read`或者访问这个Service的VIP。这样读请求就可以被转发到任意一个MySQL的主节点或者从节点。
2. 所有用户的写请求，则必须直接以DNS的方式访问到MySQL的主节点，也就是`mysql-0.mysql`这条DNS记录。

> **Kubernetes中所有的Service和pod对象，都会被自动分配同名的DNS记录**。

#### 0.4.2.2. 问题二：主从节点需要传输备份文件

推荐的做法：先搭建框架，再完善细节。其中Pod部分如何定义，是完善细节时的重点。

创建StatefulSet对象的大致框架如下：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
      - name: clone-mysql
      containers:
      - name: mysql
      - name: xtrabackup
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

首先定义一些通用的字段：

1. selector，表示这个StatefulSet要管理的Pod必须携带app=mysql这个label
2. serviceName，声明这个StatefulSet要使用的Headless Servie的名字是mysql
3. replicas，表示这个StatefulSet定义的MySQL集群有三个节点（一个主节点两个从节点）
4. volumeClaimTemplate（PVC模板），还需要管理存储状态，通过PVC模板来为每个Pod创建PVC

> StatefulSet管理的“有状态应用”的多个实例，也是通过同一份Pod模板创建出来的，使用同样的Docker镜像。**这就意味着，如果应用要求不同类型节点的镜像不一样，那就不能再使用StatefulSet，应该考虑使用Operator**。

重点就是Pod部分的定义，也就是StatefulSet的`template`字段。

**StatefulSet管理的Pod都来自同一个镜像，编写Pod时需要分别考虑这个pod的Master节点做什么，Slave节点做什么**。

##### 0.4.2.2.1. 第一步，从ConfigMap中，获取MySQL的Pod对应的配置文件

需要根据主从节点不同的角色进行相应的初始化操作，为每个Pod分配对应的配置文件。**MySQL要求集群中的每个节点都要唯一的ID文件（server-id.cnf）**。

初始化操作使用InitContainer完成，定义如下：

```yaml
      ...
      # template.spec
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
        - bash
        - "-c"
        - |
          set -ex
          # 从 Pod 的序号，生成 server-id
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # 由于 server-id=0 有特殊含义，我们给 ID 加一个 100 来避开它
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # 如果 Pod 序号是 0，说明它是 Master 节点，从 ConfigMap 里把Master 的配置文件拷贝到 /mnt/conf.d/ 目录；
          # 否则，拷贝 Slave 的配置文件
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
```

这个初始化容器主要完成的初始化操作为：

1. 从Pod的hostname里，读取到了Pod的序号，以此作为MySQL节点的server-id。
2. 通过这个序号判断当前Pod的角色（序号为0表示为Master，其他为Slave），从而把对应的配置文件从`/mnt/config-map`目录拷贝到`/mnt/conf.d`目录下。

其中文件拷贝的源目录`/mnt/config-map`，就是CongifMap在这个Pod的Volume，如下所示：

```yaml
      ...
      # template.spec
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
```

通过这个定义，init-mysql在声明了挂载config-map这个Volume之后，ConfigMap里保存的内容，就会以文件的方式出现在它的`/mnt/config-map`目录当中。

而文件拷贝的目标目录，即容器里的`/mnt/conf.d/`目录，对应的则是一个名叫conf的emptyDir类型的Volume。基于Pod Volume 共享的原理，当InitContainer复制完配置文件退出后，后面启动的MySQL容器只需要直接声明挂载这个名叫conf的Volume，它所需要的.cnf 配置文件已经出现在里面了。

##### 0.4.2.2.2. 第二步：在Slave Pod启动前，从Master或其他Slave里拷贝数据库数据到自己的目录下

再定义一个初始化容器来完成这个操作：

```yaml
      ...
      # template.spec.initContainers
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # 拷贝操作只需要在第一次启动时进行，所以如果数据已经存在，跳过
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Master 节点 (序号为 0) 不需要做这个操作
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # 使用 ncat 指令，远程地从前一个节点拷贝数据到本地
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # 执行 --prepare，这样拷贝来的数据就可以用作恢复了
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
```

这个初始化容器使用xtrabackup镜像（安装了xtrabackup工具），主要进行如下操作：

1. 在它的启动命令里首先进行判断，当初始化所需的数据（`/var/lib/mysql/mysql`目录）已经存在，或者当前Pod是Master时，不需要拷贝操作。
2. 使用Linux自带的ncat命令，向DNS记录为“mysql- <当前序号减1>.mysql”的Pod（即当前Pod的前一个Pod），发起数据传输请求，并且直接使用xbstream命令将收到的备份数据保存在`/var/lib/mysql`目录下。传输数据的方式包括scp、rsync等。
3. 拷贝完成后，初始化容器还需要对`/var/lib/mysql`目录执行`xtrabackup --prepare`命令，目的是保证拷贝的数据进入一致性状态，这样数据才能被用作数据恢复。

> 3307是一个特殊的端口，运行着一个专门负责备份MySQL数据的辅助进程。

**这个容器的`/var/lib/mysql`目录，实际上是一个名为data的PVC**。这就保证哪怕宿主机服务器宕机，数据库的数据也不会丢失。

因为Pod的Volume是被Pod中的容器所共享的，所以后面启动的MySQL容器，就可以把这个Volume挂载到自己的`/var/lib/mysql`目录下，直接使用里面的备份数据进行恢复操作。

> 通过两个初始化容器完成了对主从节点配置文件的拷贝，主从节点间备份数据的传输操作。

**注意，StatefulSet里面的所有Pod都来自同一个Pod模板，所以在定义MySQL容器的启动命令时，需要区分Master和Slave节点的不同情况**。

1. 直接启动Master角色没有问题
2. 第一次启动的Slave角色，在执行MySQL启动命令之前，需要使用初始化容器拷贝的数据进行容器的初始化操作
3. 容器是单进程模型，Slave角色的MySQL启动前，谁负责执行初始化SQL语句？

##### 0.4.2.2.3. 第三步，Slave角色的MySQL容器启动前，执行初始化SQL语句

为这个MySQL容器定义一个额外的sidecar容器，来完成初始化SQL语句的操作：

```yaml
      ...
      # template.spec.containers
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql
          # 从备份信息文件里读取 MASTER_LOG_FILEM 和 MASTER_LOG_POS 这两个字段的值，用来拼装集群初始化 SQLA
          if [[ -f xtrabackup_slave_info ]]; then
            # 如果 xtrabackup_slave_info 文件存在，说明这个备份数据来自于另一个 Slave 节点。
            # 这种情况下，XtraBackup 工具在备份的时候，就已经在这个文件里自动生成了 "CHANGE MASTER TO" SQL 语句。
            # 所以，我们只需要把这个文件重命名为 change_master_to.sql.in，后面直接使用即可
            mv xtrabackup_slave_info change_master_to.sql.in
            # 所以，也就用不着 xtrabackup_binlog_info 了
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # 如果只存在 xtrabackup_binlog_inf 文件，那说明备份来自于 Master 节点，
            # 我们就需要解析这个备份信息文件，读取所需的两个字段的值
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm xtrabackup_binlog_info
            # 把两个字段的值拼装成 SQL，写入 change_master_to.sql.in 文件
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi
          # 如果 change_master_to.sql.in，就意味着需要做集群初始化工作
          if [[ -f change_master_to.sql.in ]]; then
            # 但一定要先等 MySQL 容器启动之后才能进行下一步连接 MySQL 的操作
            echo "Waiting for mysqld to be ready (accepting connections)"
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done
            echo "Initializing replication from clone position"
            # 将文件 change_master_to.sql.in 改个名字，防止这个 Container 重启的时候，
            # 因为又找到了 change_master_to.sql.in，从而重复执行一遍这个初始化流程
            mv change_master_to.sql.in change_master_to.sql.orig
            # 使用 change_master_to.sql.orig 的内容，
            # 也是就是前面拼装的 SQL，组成一个完整的初始化和启动 Slave 的 SQL 语句
            mysql -h 127.0.0.1 <<EOF
          $(<change_master_to.sql.orig),
            MASTER_HOST='mysql-0.mysql',
            MASTER_USER='root',
            MASTER_PASSWORD='',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
          fi
          # 使用 ncat 监听 3307 端口。它的作用是，在收到传输请求的时候，
          # 直接执行 "xtrabackup --backup" 命令，备份 MySQL 的数据并发送给请求者
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
```

在这个sidecar容器的启动命令中，完成两部分工作。

###### 0.4.2.2.3.1. 工作一：MySQL节点初始化

这个初始化需要的SQL是sidecar容器拼装出来、保存在名为change_master_to.sql.in的文件里的。具体过程如下：

1. sidecar容器首先判断当前Pod的`/var/lib/mysql`目录下，是否有`xtrabackup_slave_info`这个备份信息文件。
    - 如果**有**，说明这个目录下的备份数据库是由一个Slave节点生成的。这种情况下，xtrabackup工具在备份的时候，就已经在这个文件里生成了`“CHANGE MASTER TO”`SQL语句。所以只需要把这个文件名重命名为`change_master_to.sql.in`，然后直接使用即可。
    - 如果**没有**，但是存在`xtrabackup_binlog_info`文件，那就说明备份数据来自Master节点。这种情况下，sidecar容器需要解析这个备份文件，读取`MASTER_LOG_FILE`和`MASTER_LOG_POS`这两个字段的值，用它们拼装出初始化SQL语句，然后把这句SQL写入`change_master_to.sql.in`文件中。

    > 只要`change_master_to.sql.in`存在，那就说明下一个步骤是进行集群初始化操作。

2. sidecar容器执行初始化操作。即，读取并执行`change_master_to.sql.in`里面的`“CHANGE MASTER TO”`SQL语句，在执行START SLAVE命令，一个Slave角色就启动成功了。

> Pod里面的容器没有先后顺序，所以在执行初始化SQL之前，必须先执行`select 1`来检查MySQL服务是否已经可用。

**当初始化操作都执行完成后，需要删除前面用到的这些备份信息文件，否则下次这个容器重启时，就会发现这些文件已经存在，然后又重新执行一次数据恢复和集群初始化的操作，这就不对了。同样的`change_master_to.sql.in`在使用后也要被重命名，以免容器重启时因为发现这个文件而又执行一遍初始化**。

###### 0.4.2.2.3.2. 工作二：启动数据传输服务

sidecar容器使用ncat命令启动一个工作在3307端口上的网络发送服务。一旦收到数据传输请求时，sidecar容器就会调用`xtrabackup --backup`命令备份当前MySQL的数据，然后把备份数据返回给请求者。

> 这就是为什么在初始化容器里面定义数据拷贝的时候，访问的是**上一个MySQL节点**的3307端口。

**sidecar容器和MySQL容器处于同一个Pod中，它们是直接通过`localhost`来访问和备份MySQL的数据的，非常方便**。数据的备份方式有多种，也可使用`innobackupex`命令。

完成上述初始化操作后，定义的MySQL容器就比较简单，如下：

```yaml
      ...
      # template.spec
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # 通过 TCP 连接的方式进行健康检查
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
```

使用MySQL官方镜像，数据目录`/var/lib/mysql`，配置文件目录`/etc/mysql/conf.d`。并且为容器定了livenessProbe，通过`mysqladmin Ping`命令来检查它是否健康。同时定义readinessProbe，通过`SQL（select 1）`来检查MySQL服务是否可用。凡是readinessProbe检查失败的Pod都会从Service中被踢除。

> 如果MySQL容器是Slave角色时，它的数据目录中的数据就是来自初始化容器从其他节点里拷贝而来的备份。它的配置目录里的内容则是是来自ConfigMap对应的Volume。它的初始化工作由sidecar容器完成。

### 0.4.3. 创建PV

使用Rook存储插件创建PV：

```yaml
kubectl create -f rook-storage.yaml

cat rook-storage.yaml

apiVersion: ceph.rook.io/v1beta1
kind: Pool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: ceph.rook.io/block
parameters:
  pool: replicapool
  clusterNamespace: rook-ceph
```

在这里使用到StorageClass来完成这个操作，它的作用是自动地为集群里存在的每个PVC调用存储插件创建对应的PV，从而省去了手动创建PV的过程。

> 在使用Rook时，在MySQL的StatefulSet清单文件中的volumeClaimTemplates字段需要加上声明storageClassName=rook-ceph-block，这样才能使用Rook提供的持久化存储。

### 0.4.4. 小结

1. “人格分裂”：在解决需求的过程中，一定要记得思考，该Pod在扮演不同角色时的不同操作。
2. “阅后即焚”：很多“有状态应用”的节点，只是在第一次启动的时候才需要做额外处理。所以，在编写YAML文件时，一定要考虑到**容器重启**的情况，不能让这一次的操作干扰到下一次容器启动。
3. “容器之间平等无序”：除非InitContainer，否则一个Pod里的多个容器之间，是完全平等的。**所以，镜像设计的sidecar，绝不能对容器的启动顺序做出假设，否则就需要进行前置检查**。

> StatefulSet就是一个特殊的Deployment，只是这个“Deployment”的每个Pod实例的名字里，都携带了一个唯一并且固定的编号。

- 这个编号的顺序，固定了Pod之间的**拓扑关系**；
- 这个编号对应的DNS记录，固定了Pod的**访问方式**；
- 这个编号对应的PV，绑定了Pod与**持久化存储**的关系。

所有，当Pod被删除重建时，这些“状态”都会保持不变。

**如果应用没办法通过上述方式进行状态的管理，就代表StatefulSet已经不能解决它的部署问题，Operator可能是一个更好的选择**。
