---
title: 03-Accessing-Dashboard
date: 2020-04-14T10:09:14.202627+08:00
draft: false
---

在集群上安装`Dashboard`后，可以通过几种不同的方式访问它。请注意，本文档未描述访问集群应用程序的所有可能方法。如果在尝试访问`Dashboard`时出现任何错误，请首先阅读[常见问题解答](/kubernetes/Dashboard/06-FQA.md)并检查已关闭的问题。在大多数情况下，错误是由群集配置问题引起的。

## 1.7.x及以上版本

**重要信息**：`HTTPS endpoints`仅在使用推荐安装程序时可用，按照[入门指南](/kubernetes/Dashboard/01-Guide.md#入门指南)部署`Dashboard`或手动提供`--tls-key-file`和`--tls-cert-file`标志。如果没有，并且想通过HTTP访问`Dashboard`，则可以使用与旧版本相同的方式访问。

> 注意：不应通过HTTP公开`Dashboard`。对于通过HTTP访问的域，将无法登录。单击登录页面上的登录按钮后不会发生任何事情。

## kubectl proxy

`kubectl proxy`在本地主机和`Kubernetes API Server`之间创建代理服务器。默认情况下，它只能在本地（从启动它的计算机）访问。

首先让检查`kubectl`是否已正确配置并且是否可以访问群集。如果出现错误，请按照[本指南](https://kubernetes.io/docs/tasks/tools/install-kubectl/)安装和设置`kubectl`。

```bash
kubectl cluster-info
# 输出如下
Kubernetes master is running at https://192.168.30.148:6443
Heapster is running at https://192.168.30.148:6443/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://192.168.30.148:6443/api/v1/namespaces/kube-system/services/kube-dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

启动本地代理服务器：

```bash
kubectl proxy
Starting to serve on 127.0.0.1:8001
```

启动代理服务器后，可以从浏览器访问Dashboard。

要访问`Dashboard`的`HTTPS endpoints`，请转到：http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

注意：不应使用`kubectl proxy`命令公开暴露`Dashboard`，因为它只允许HTTP连接。对于`localhost`和`127.0.0.1`以外的域，将无法登录。单击登录页面上的登录按钮后不会发生任何事情。

## NodePort

这种访问`Dashboard`的方式仅建议用于**单节点**设置中的开发环境。

编辑`kubernetes-dashboard service`：

```bash
kubectl -n kube-system edit service kubernetes-dashboard
```

可以看到服务的`yaml`表示，将`type:ClusterIP`修改为`type:NodePort`并保存文件。如果已经更改，请转到下一步。

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
...
  name: kubernetes-dashboard
  namespace: kube-system
  resourceVersion: "343478"
  selfLink: /api/v1/namespaces/kube-system/services/kubernetes-dashboard-head
  uid: 8e48f478-993d-11e7-87e0-901b0e532516
spec:
  clusterIP: 10.100.124.90
  externalTrafficPolicy: Cluster
  ports:
  - port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: ClusterIP       ## 修改这里
status:
  loadBalancer: {}
```

接下来，检查`Dashboard`暴露的端口。

```bash
kubectl -n kube-system get service kubernetes-dashboard

NAME                   CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes-dashboard   10.100.124.90   <nodes>       443:31707/TCP   21h
```

`Dashboard`已在端口31707（HTTPS）上公开。现在，可以从以下浏览器访问它：`https://<master-ip>:31707`。通过执行`kubectl cluster-info`可以找到`master-ip`。通常它是机器的`127.0.0.1`或`IP`，假设集群直接在服务器上运行，执行这些命令。

如果尝试在多节点群集上使用`NodePort`公开`Dashboard`，则必须找到运行`Dashboard`的节点的`IP`才能访问它。应该访问`https://<node-ip>:<nodePort>`，而不是访问`https://<master-ip>:<nodePort>`。

## API Server

如果`Kubernetes API Server`公开并可从外部访问，可以直接访问`Dashboard`: `https://<master-ip>:<apiserver-port>/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`

> 注意：只有选择在浏览器中安装用户证书时，才能使用这种访​​问`Dashboard`的方式。在示例中，可以使用`kubeconfig`文件用于联系`API Server`的证书。

## Ingress

`Dashboard`也可以使用`Ingress`资源公开。有关更多信息，请查看这里：

https://kubernetes.io/docs/concepts/services-networking/ingress