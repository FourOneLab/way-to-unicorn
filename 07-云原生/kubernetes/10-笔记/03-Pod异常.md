---
title: 03 pod异常
date: 2020-04-14T10:09:14.202627+08:00
draft: false
---

- [0.1. 常用命令行工具](#01-常用命令行工具)
- [0.2. Pod 一直处于 Pending 状态](#02-pod-一直处于-pending-状态)
- [0.3. Pod 处于 Waiting 或 ContainerCreating 状态](#03-pod-处于-waiting-或-containercreating-状态)
- [0.4. Pod 处于 ImagePullBackOff 状态](#04-pod-处于-imagepullbackoff-状态)
  - [0.4.1. 私有镜像](#041-私有镜像)
- [0.5. Pod 一直处于 CrashLoopBackOff 状态](#05-pod-一直处于-crashloopbackoff-状态)
- [0.6. Pod 处于 Error 状态](#06-pod-处于-error-状态)
- [0.7. Pod 处于 Terminating 或 Unknown 状态](#07-pod-处于-terminating-或-unknown-状态)
- [0.8. Pod 行为异常](#08-pod-行为异常)
- [0.9. 修改静态 Pod 的 Manifest 后未自动重建](#09-修改静态-pod-的-manifest-后未自动重建)

## 0.1. 常用命令行工具

工具| 作用 | 例子
---|---|---|
kubectl|用于查看 Kubernetes 集群以及容器的状态|如 kubectl describe pod `<pod-name>`
journalctl|用于查看 Kubernetes 组件日志|如 journalctl -u kubelet -l
iptables和ebtables|用于排查 Service 是否工作|如 iptables -t nat -nL 查看 kube-proxy 配置的 iptables 规则是否正常
tcpdump|用于排查容器网络问题|如 tcpdump -nn host 10.240.0.8
perf|Linux 内核自带的性能分析工具|常用来排查性能问题

一般来说，无论 Pod 处于什么异常状态，都可以执行以下命令来查看 Pod 的状态：

```bash
//查看 Pod 的配置是否正确
kubectl get pod <pod-name> -o yaml

//查看 Pod 的事件
kubectl describe pod <pod-name>

//查看容器日志
kubectl logs <pod-name> [-c <container-name>]
```

**这些事件和日志通常都会有助于排查 Pod 发生的问题。**

## 0.2. Pod 一直处于 Pending 状态

Pending 说明 Pod 还没有调度到某个 Node 上面。

可以通过 `kubectl describe pod <pod-name>` 命令查看到当前 Pod 的事件，进而判断为什么没有调度。

**可能的原因:**

- 资源不足，集群内所有的 Node 都不满足该 Pod 请求的 CPU、内存、GPU 等资源
- HostPort 已被占用，通常推荐使用 Service 对外开放服务端口

## 0.3. Pod 处于 Waiting 或 ContainerCreating 状态

首先还是通过 `kubectl describe pod <pod-name>` 命令查看到当前 Pod 的事件。

**可能的原因:**

- 镜像拉取失败，比如配置了错误的镜像
- Kubelet 无法访问镜像（国内环境访问 gcr.io 需要特殊处理）
- 私有镜像的密钥配置错误
- 镜像太大，拉取超时（可以适当调整 kubelet 的 `--image-pull-progress-deadline` 和 `--runtime-request-timeout` 选项）
- CNI 网络错误，一般需要检查 CNI 网络插件的配置，比如
  - 无法配置 Pod 网络
  - 无法分配 IP 地址
- 容器无法启动，需要检查是否打包了正确的镜像或者是否配置了正确的容器参数

## 0.4. Pod 处于 ImagePullBackOff 状态

这通常是镜像名称配置错误或者私有镜像的密钥配置错误导致。

这种情况可以使用 docker pull `<image>` 来验证镜像是否可以正常拉取。

### 0.4.1. 私有镜像

1. 首先创建一个 docker-registry 类型的 Secret

    ```bash
    kubectl create secret docker-registry my-secret \
    --docker-server=DOCKER_REGISTRY_SERVER \
    --docker-username=DOCKER_USER \
    --docker-password=DOCKER_PASSWORD \
    --docker-email=DOCKER_EMAIL
    ```

2. 然后在容器中引用这个 Secret

    ```bash
    spec:
    containers:
    - name: private-reg-container
    image: <your-private-image>
    imagePullSecrets:
    - name: my-secret
    ```

## 0.5. Pod 一直处于 CrashLoopBackOff 状态

CrashLoopBackOff 状态说明容器曾经启动了，但又异常退出了。

此时可以先查看一下容器的日志

```bash
kubectl logs <pod-name>
kubectl logs --previous <pod-name>
```

这里可以发现一些容器退出的原因，比如:

- 容器进程退出
- 健康检查失败退出

此时如果还未发现线索，还可以到容器内执行命令来进一步查看退出原因

```bash
kubectl exec cassandra -- cat /var/log/cassandra/system.log
```

如果还是没有线索，那就需要 SSH 登录该 Pod 所在的**Node**上，查看 Kubelet 或者 Docker 的日志进一步排查了

查询 Node

```bash
kubectl get pod <pod-name> -o wide
```

## 0.6. Pod 处于 Error 状态

通常处于 Error 状态说明 Pod **启动过程中发生了错误**。

常见的原因:

- 依赖的 ConfigMap、Secret 或者 PV 等不存在
- 请求的资源超过了管理员设置的限制，比如超过了 LimitRange 等
- 违反集群的安全策略，比如违反了 PodSecurityPolicy 等
- 容器无权操作集群内的资源，比如开启 RBAC 后，需要为 ServiceAccount配置角色绑定

## 0.7. Pod 处于 Terminating 或 Unknown 状态

从 v1.5 开始，Kubernetes 不会因为 Node 失联而删除其上正在运行的 Pod，而是将其标记为 Terminating 或 Unknown 状态。

想要删除这些状态的 Pod 有三种方法：

- （1）从集群中删除该 Node。
  - 使用公有云时，kube-controller-manager 会在 VM 删除后自动删除对应的 Node。
  - 而在物理机部署的集群中，需要管理员手动删除 Node（如 `kubectl delete node <node-name>`。

- （2）Node 恢复正常。Kubelet 会重新跟 kube-apiserver 通信确认这些 Pod 的期待状态，进而再决定删除或者继续运行这些 Pod。

- （3）用户强制删除。用户可以执行 `kubectl delete pods <pod> --grace-period=0 --force` 强制删除 Pod。除非明确知道 Pod 的确处于停止状态（比如 Node 所在 VM 或物理机已经关机），否则不建议使用该方法。**特别是 StatefulSet 管理的 Pod，强制删除容易导致脑裂或者数据丢失等问题。**

## 0.8. Pod 行为异常

这里所说的行为异常是指 **Pod没有按预期的行为执行**。

**比如没有运行 podSpec 里面设置的命令行参数。这一般是 podSpec yaml 文件内容有误。**

1. 可以尝试使用 --validate 参数重建容器

    ```bash
    kubectl delete pod mypod
    kubectl create --validate -f mypod.yaml
    ```

2. 也可以查看创建后的 podSpec 是否正确

    ```bash
    kubectl get pod mypod -o yaml
    ```

## 0.9. 修改静态 Pod 的 Manifest 后未自动重建

Kubelet 使用 inotify 机制检测 `/etc/kubernetes/manifests` 目录（可通过 Kubelet 的 `--pod-manifest-path` 选项指定）中静态 Pod 的变化，并在文件发生变化后重新创建相应的 Pod。

但有时也会发生修改静态 Pod 的 Manifest 后未自动创建新 Pod 的情景，此时一个简单的修复方法是重启 Kubelet。
