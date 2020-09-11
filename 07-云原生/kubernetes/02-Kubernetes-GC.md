# kubernetes垃圾回收

> **垃圾回收（GC）就是从系统中删除未使用的对象，并释放分配给它们的计算资源**。GC 存在于所有的高级编程语言中，较低级的编程语言通过系统库实现 GC。

## OwnerReference

在面向对象的语言中，一些对象会引用其他对象或者直接由其他对象组成，Kubernetes也有类似形式，例如Deployment管理一组ReplicaSet，ReplicaSet管理一组Pod。

Kubernetes对象的定义中，没有明确所有者之间的关系，每个从属对象都具有唯一数据字段名称 `metadata.ownerReferences` 用于确定关系。

> 从 Kubernetes v1.8 开始，对于 ReplicaSet、StatefulSet、DaemonSet、Deployment、Job、 CronJob 等创建或管理的对象，会自动为其设置 ownerReferences 的值，也可以手动设置。

```bash
sugoi@sugoi:~$ kubectl -n university get po portal-0 -o json | jq ".metadata.ownerReferences"
[
  {
    "apiVersion": "apps/v1",
    "blockOwnerDeletion": true,
    "controller": true,
    "kind": "StatefulSet",
    "name": "portal",
    "uid": "239e3b00-8b66-41b9-b6e5-39d4d07fa567"
  }
]
```

在Kubernetes中对象之间的依赖与对象之间的管理关系是正好相反的。

- 对象管理：Deployment--->ReplicaSet--->Pod
- 对象依赖：Pod---> ReplicaSet--->Deployment

Kuberetes 社区引入并实现了 Garbage Collector Controller（垃圾回收器），用更好用且更简单的方式实现 GC，分为两大类 GC：

- 级联（Cascading）：所有者被删除，集群中的从属对象也会被删除
- 孤儿（Orphan）：所有者被删除，集群中的从属对象处于“孤儿”状态

## 级联删除

在级联删除（cascading deletion strategy）中，从属对象（dependent object）与所有者对象（owner object）会被一起删除，又分为两种模式：前台（foreground）和后台（background）。

- 前台级联删除（Foreground Cascading Deletion）：所有者对象的删除将会持续到其所有从属对象都被删除为止。当所有者被删除时，会进入“正在删除”（deletion in progress）状态，此时：
  - 对象仍然可以通过 REST API 查询到（kubectl/kuboard）
  - 对象的 `deletionTimestamp` 字段被设置
  - 对象的 `metadata.finalizers` 包含值 `foregroundDeletion`

> 一旦对象被设置为 “正在删除” 状态，垃圾回收器将删除其从属对象。当垃圾回收器已经删除了所有的“blocking”从属对象（`ownerReference.blockOwnerDeletion=true`的对象）以后，将删除所有者对象。

- 后台级联删除（Background Cascading Deletion）：立即删除所有者的对象，并由垃圾回收器在后台删除其从属对象，比前台级联删除快，不用等待时间来删除从属对象。

## 孤儿删除

在孤儿删除策略（orphan deletion strategy）中，会直接删除所有者对象，并将从属对象中的 ownerReference 元数据设置为**默认值**。之后垃圾回收器会确定孤儿对象并将其删除。

## 垃圾收集器工作原理

如果对象的 OwnerReferences 元数据中没有任何所有者对象，那么垃圾回收器会删除该对象。垃圾回收器由 Scanner、Garbage Processor 和 Propagator 组成：

- Scanner：它会检测 K8s 集群中支持的所有资源，并通过控制循环周期性地检测。它会扫描系统中的所有资源，并将每个对象添加到"脏队列"（dirty queue）中。
- Garbage Processor：它由在"脏队列"上工作的 worker 组成。每个 worker 都会从"脏队列"中取出对象，并检查该对象里的 OwnerReference 字段是否为空。如果为空，那就从“脏队列”中取出下一个对象进行处理；如果不为空，它会检测 OwnerReference 字段内的 owner resoure object 是否存在，如果不存在，会请求 API 服务器删除该对象。
- Propagator ：用于优化垃圾回收器，它包含以下三个组件：
  - EventQueue：负责存储 k8s 中资源对象的事件
  - DAG（有向无环图）：负责存储 k8s 中所有资源对象的 owner-dependent 关系
  - Worker：从 EventQueue 中取出资源对象的事件，并根据事件的类型会采取操作

有了 Propagator 的加入后，可以仅在 GC 开始运行的时候，让 Scanner 扫描系统中所有的对象，然后将这些信息传递给 Propagator 和“脏队列”。只要 DAG 一建立起来之后，那么 Scanner 其实就没有再工作的必要了。
