# 入门指南

`Kubernetes Dashboard`是一个通用的基于`WebUI`的`kubernetes`集群管理工具。用户管理运行在集群中的应用并对这些运行的应用进行快速的故障排查，同时也可以用于管理集群本身。

> **重要信息**：在执行任何后续步骤之前，请阅读“[访问控制](/kubernetes/Dashboard/02-Access-Control.md)”指南。默认的`Dashboard`部署包含运行所需的最小`RBAC`权限集。

执行以下命令部署`Dashboard`：

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

[YAML](/kubernetes/Dashboard/kubernetes-dashboard.yaml)文件参考这里。

要从本地工作站访问`Dashboard`，必须为`Kubernetes`集群创建安全通道。运行以下命令：

```bash
kubectl proxy
```

通过以下地址访问`Dashboard`：<http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/>

## 创建一个认证令牌（RBAC）

要了解如何创建示例用户并登录，请按照[创建示例用户指南](/kubernetes/Dashboard/04-Creating-sample-user.md)进行操作。

注意：

- `Kubeconfig`身份验证方法不支持外部身份提供程序或基于证书的身份验证。
- `Dashboard`只能通过HTTPS访问。
- `Heapster`必须在集群中运行才能获取指标并以提醒方式展示。在“[集成](/kubernetes/Dashboard/05-Integrations.md)”指南中阅读更多相关信息。

## 访问控制

当`Dashboard`安装完成并且能够正常访问后，将专注于为用户配置对群集资源的访问控制。从版本`1.7`开始：

- `Dashboard`不再具有默认授予的**完全管理员权限**。
- 所有权限都被撤销，并且只授予了使`Dashboard`工作所需的最小权限。

> **重要信息**：本说明仅针对使用`Dashboard 1.7`及更高版本的用户。如果`Dashboard`只能由受信任的人员访问，所有人员都具有完全管理员权限，那么可以授予他们集群的管理员权限。请注意，其他应用程序不应直接访问`Dashboard`，因为它可能会导致权限提升。确保集群内流量仅限于命名空间，或者只是撤消对集群内其他应用程序的`Dashboard`访问权限。

## 简介

Kubernetes支持几种验证和授权用户的方法。可以在[这里](https://kubernetes.io/docs/admin/authentication)和[这里](https://kubernetes.io/docs/admin/authorization)阅读它们。授权由`Kubernetes API Server`处理。`Dashboard`仅充当代理并将所有身份验证信息传递给它。在禁止访问的情况下，相应的警告将显示在`Dashboard`中。

## Dashboard默认权限

### v1.7

- `create`和`watch`在`kube-system`命名空间中`secret`的权限，以创建和监视`kubernetes-dashboard-key-holder` `secret`的更改。
- `get`，`update`和`delete`在`kube-system`命名空间中名为`kubernetes-dashboard-key-holder`和`kubernetes-dashboard-certs`的`secret`的权限。
- 对`kube-system`命名空间中的`heapster`服务的代理权限，以允许从`heapster`获取指标。

### v1.8

- 为`kube-system`命名空间中的`secret`创建权限以使其能够创建`kubernetes-dashboard-key-holder`的`secret`。
- `get`，`update`和`delete`在`kube-system`命名空间中名为`kubernetes-dashboard-key-holder`和`kubernetes-dashboard-certs`的`secret`的权限。
- `get`并`update`在`kube-system`命名空间中名为`kubernetes-dashboard-settings`的`configmap`的权限。
- 对`kube-system`命名空间中的`heapster`服务的代理权限，以允许从`heapster`获取指标。

## 认证

从版本`1.7`开始，`Dashboard`支持基于以下内容的用户身份验证：

- `Authorization: Bearer <token>`：在每次向`Dashboard`发出请求时传递请求头。从`1.6`版开始支持。具有最高优先级。如果存在，则不会显示登录界面。
- `Bearer Token`：在`Dashboard`登录界面中使用。
- 用户名/密码：在`Dashboard`登录界面中使用。
- `kubeconfig`：在`Dashboard`登录界面中使用。

### 登录界面

登录界面在`1.7`版中引入。如果使用的是最新推荐的安装，则默认情况下将启用登录功能。在任何其他情况下，如果更喜欢手动配置证书，则需要将`--tls-cert-file`和`--tls-cert-key`标志传递给`Dashboard`。HTTPS端点将在`Dashboard`容器的端口`8443`上公开。可以通过提供`--port`标志来更改它。

使用“跳过”选项将使`Dashboard`使用`Dashboard Service Account`的权限。默认情况下，跳过按钮自`1.10.1`起禁用。使用`--enable-skip-login dashboard`标志显示它。

![image](/images/login-view.png)

### Authorization header

在通过HTTP访问`Dashboard`时，使用`Authorization header`是使`Dashboard`充当用户的唯一方法。请注意，由于普通HTTP流量容易受到`MITM`攻击（中间人攻击），因此存在一些风险。

要使`Dashboard`使用`Authorization header`，只需将每个请求中的`Authorization：Bearer <token>`传递给`Dashboard`。这可以通过在`Dashboard`前配置反向代理来实现。代理服务器将负责身份提供者的身份验证，并将请求头中生成的令牌传递给`Dashboard`。请注意，需要正确配置`Kubernetes API Server`才能接受这些令牌。

要快速测试它，请查看允许手动修改请求头的[Requestly](https://chrome.google.com/webstore/detail/requestly-redirect-url-mo/mdnleldcmiljblolnjhpnblkcekpdkpa)浏览器插件。

> 重要信息：如果通过`API Server`代理访问`Dashboard`，则`Authorization header`将不起作用。访问`Dashboard`指南中描述的`kubectl proxy`和`API Server访问`Dashboard`的方式都不起作用。这是因为一旦请求到达`API Server`，所有其他标头都将被删除。

### Bearer Token

建议首先熟悉[`Kubernetes身份验证`](https://kubernetes.io/docs/admin/authentication)文档，以了解如何获取可用于登录的令牌。例如，每个`Service Account`都有一个具有有效`Bearer Token`的`secret`，可用于登录`Dashboard`。

#### Bearer Token 样例

![images](/images/Bearer-Token.png)

#### 使用kubectl获取token

默认情况下，在`Kubernetes`中创建了许多`Service Account`，它们都具有不同的访问权限。为了找到任何可用于登录的令牌，使用`kubectl`：

```bash
# 检查kube-system命名空间中的现有secret
kubectl -n kube-system get secret
# 所有类型为'kubernetes.io/service-account-token'的secret都将允许登录
# 请注意，它们具有不同的权限
NAME                                     TYPE                                  DATA      AGE
attachdetach-controller-token-xw1tw      kubernetes.io/service-account-token   3         10d
bootstrap-signer-token-gz8qp             kubernetes.io/service-account-token   3         10d
bootstrap-token-f46476                   bootstrap.kubernetes.io/token         5         10d
certificate-controller-token-tp34m       kubernetes.io/service-account-token   3         10d
daemon-set-controller-token-fqvwx        kubernetes.io/service-account-token   3         10d
default-token-3d5t4                      kubernetes.io/service-account-token   3         10d
deployment-controller-token-3gd7d        kubernetes.io/service-account-token   3         10d
disruption-controller-token-gdsxq        kubernetes.io/service-account-token   3         10d
endpoint-controller-token-vrxpg          kubernetes.io/service-account-token   3         10d
flannel-token-xrldr                      kubernetes.io/service-account-token   3         10d
foo-secret                               kubernetes.io/tls                     2         6d
generic-garbage-collector-token-hk04n    kubernetes.io/service-account-token   3         10d
heapster-token-wgwgx                     kubernetes.io/service-account-token   3         10d
horizontal-pod-autoscaler-token-4865f    kubernetes.io/service-account-token   3         10d
job-controller-token-q0wdp               kubernetes.io/service-account-token   3         10d
kd-dashboard-token-token-bw08g           kubernetes.io/service-account-token   3         7d
kube-dns-token-qc79f                     kubernetes.io/service-account-token   3         10d
kube-proxy-token-22kd5                   kubernetes.io/service-account-token   3         10d
kubernetes-dashboard-head-token-pq4hk    kubernetes.io/service-account-token   3         6d
kubernetes-dashboard-key-holder          Opaque                                2         5d
kubernetes-dashboard-token-7qmbc         kubernetes.io/service-account-token   3         6d
namespace-controller-token-k6zfw         kubernetes.io/service-account-token   3         10d
node-controller-token-0821f              kubernetes.io/service-account-token   3         10d
persistent-volume-binder-token-vgt06     kubernetes.io/service-account-token   3         10d
pod-garbage-collector-token-6pz9t        kubernetes.io/service-account-token   3         10d
replicaset-controller-token-kzpmc        kubernetes.io/service-account-token   3         10d
replication-controller-token-4x5wh       kubernetes.io/service-account-token   3         10d
resourcequota-controller-token-srbqv     kubernetes.io/service-account-token   3         10d
service-account-controller-token-7qp8r   kubernetes.io/service-account-token   3         10d
service-controller-token-p46zd           kubernetes.io/service-account-token   3         10d
statefulset-controller-token-npt26       kubernetes.io/service-account-token   3         10d
token-cleaner-token-gdfq3                kubernetes.io/service-account-token   3         10d
ttl-controller-token-pt064               kubernetes.io/service-account-token   3         10d
# 从'replicaset-controller-token-kzpmc'获取令牌，它应该具有查看群集中的副本集的权限
kubectl -n kube-system describe secrets replicaset-controller-token-kzpmc

Name:		replicaset-controller-token-kzpmc
Namespace:	kube-system
Labels:		<none>
Annotations:	kubernetes.io/service-account.name=replicaset-controller
		kubernetes.io/service-account.uid=d0d93741-96c5-11e7-8245-901b0e532516

Type:	kubernetes.io/service-account-token

Data
====
ca.crt:		1025 bytes
namespace:	11 bytes
token: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJyZXBsaWNhc2V0LWNvbnRyb2xsZXItdG9rZW4ta3pwbWMiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoicmVwbGljYXNldC1jb250cm9sbGVyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiZDBkOTM3NDEtOTZjNS0xMWU3LTgyNDUtOTAxYjBlNTMyNTE2Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOnJlcGxpY2FzZXQtY29udHJvbGxlciJ9.wAzSLnDudLbAeeU9eRsPlVELLJ6EokDT8mLpEm868hh_Ot_7Pp68321a9ssddIczPqJeifG_nFaKovkdmjozysLvwYCfpKZEkQjl_VKrr9oBzrlGUk-1oaztoLMxxtlm1FuVd_2moWbrQYRv0sGdtmZLZl9vAfW2s3vpwSn_t8XRB-bcombjBakbjm3os4RsiutAS7vevdyJMkAIjKalwZnNJ0nMaY8qEpA85WjEF86zjj_QBpZFt8tJZ7IO3uUuTLgBdDJr8dPwJhkMZp_eE_zGkpsBlp34fdg-1_TQGDm0fokvBkRt8luSR9HnyYxk6UEk5MT60WeaEzvCe3J4SA
```

现在可以使用显示的令牌登录`Dashboard`。要了解有关如何配置和使用`Bearer Token`的更多信息，请阅读“[简介](#简介)”部分。

### Basic

Basic认证默认是关闭的，原因是`Kubernetes API Server`需要配置授权模式`ABAC`和`--basic-auth-file`标志。如果`API Server`没有自动回退到[匿名用户](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#anonymous-requests)，那么无法检查提供的凭据是否有效。

为了在`Dashboard`中启用Basic认证，必须提供`authentication-mode=basic`标志。默认情况下，它设置为`--authentication-mode=token`。

### Kubeconfig

为方便起见，提供了这种登录方法。`kubeconfig`文件仅支持`-authentication-mode`标志指定的身份验证选项。如果它配置为使用任何其他方式，错误将显示在`Dashboard`中。目前不支持外部身份提供程序或基于证书的身份验证。

## 管理员权限

> 重要提示：在继续操作之前，请确保知道自己在做什么。向`Dashboard`的`Service Account`授予管理员权限可能存在安全风险。

可以通过创建以下`ClusterRoleBinding`来授予`Dashboard`的`Service Accout`完全管理员权限。根据所选的安装方法复制YAML文件并另存为，如`dashboard-admin.yaml`。使用`kubectl create -f dashboard-admin.yaml`进行部署。之后，可以使用登录页面上的“跳过”选项来访问`Dashboard`。

### 官方版

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
```

### 开发版

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-head
  labels:
    k8s-app: kubernetes-dashboard-head
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-head
  namespace: kube-system
```

## 访问 Dashboard

### 1.7.x及以上版本

**重要信息**：`HTTPS endpoints`仅在使用推荐安装程序时可用，按照[入门指南](/kubernetes/Dashboard/01-Guide.md#入门指南)部署`Dashboard`或手动提供`--tls-key-file`和`--tls-cert-file`标志。如果没有，并且想通过HTTP访问`Dashboard`，则可以使用与旧版本相同的方式访问。

> 注意：不应通过HTTP公开`Dashboard`。对于通过HTTP访问的域，将无法登录。单击登录页面上的登录按钮后不会发生任何事情。

### kubectl proxy

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

要访问`Dashboard`的`HTTPS endpoints`，请转到：<http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/>

注意：不应使用`kubectl proxy`命令公开暴露`Dashboard`，因为它只允许HTTP连接。对于`localhost`和`127.0.0.1`以外的域，将无法登录。单击登录页面上的登录按钮后不会发生任何事情。

### NodePort

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

### API Server

如果`Kubernetes API Server`公开并可从外部访问，可以直接访问`Dashboard`: `https://<master-ip>:<apiserver-port>/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`

> 注意：只有选择在浏览器中安装用户证书时，才能使用这种访​​问`Dashboard`的方式。在示例中，可以使用`kubeconfig`文件用于联系`API Server`的证书。

### Ingress

`Dashboard`也可以使用`Ingress`资源公开。有关更多信息，请查看这里：

<https://kubernetes.io/docs/concepts/services-networking/ingress>

## 创建简单用户

在本指南中，将了解如何使用`Kubernetes`的`Service Account`机制创建新用户，授予此用户管理员权限并使用与此用户关联的`Bearer Token`登录仪表板。

将下面提供的的代码片段复制到某个`dashboard-adminuser.yaml`文件，并使用`kubectl apply -f dashboard-adminuser.yaml`创建它们。

### 创建 Service Account

首先在`kube-system`命名空间中创建名为`admin-user`的`Service Account`。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```

### 创建 ClusterRoleBinding

在大多数情况下，使用`kops`或`kubeadm`或任何其他流行工具配置集群，`ClusterRole` `admin-Role`已存在于集群中。可以使用它并为`ServiceAccount`仅创建`ClusterRoleBinding`。

> 注意：`ClusterRoleBinding`资源的`apiVersion`可能在`Kubernetes`版本之间有所不同。在`Kubernetes v1.8`之前，`apiVersion`是`rbac.authorization.k8s.io/v1beta1`。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

### Bearer Token

现在需要找到可以用来登录的令牌。执行以下命令：

```bash
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')

# 它应该打印如下：

Name:         admin-user-token-6gl6l
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=b16afba9-dfec-11e7-bbb9-901b0e532516

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTZnbDZsIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJiMTZhZmJhOS1kZmVjLTExZTctYmJiOS05MDFiMGU1MzI1MTYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.M70CU3lbu3PP4OjhFms8PVL5pQKj-jj4RNSLA4YmQfTXpPUuxqXjiTf094_Rzr0fgN_IVX6gC4fiNUL5ynx9KU-lkPfk0HnX8scxfJNzypL039mpGt0bbe1IXKSIRaq_9VW59Xz-yBUhycYcKPO9RM2Qa1Ax29nqNVko4vLn1_1wPqJ6XSq3GYI8anTzV8Fku4jasUwjrws6Cn6_sPEGmL54sq5R4Z5afUtv-mItTmqZZdxnkRqcJLlg2Y8WbCPogErbsaCDJoABQ7ppaqHetwfM_0yMun6ABOQbIwwl8pspJhpplKwyo700OSpvTT9zlBsu-b35lzXGBRHzv5g_RA
```

现在复制令牌并将其粘贴到登录屏幕上的`Enter token`字段中。

![image](/images/Enter-Token.png)

单击登录按钮就可以了。现在以管理员身份登录。

![image](/images/admin.png)

要了解有关如何在`Kubernetes`中授予/拒绝权限的更多信息，请阅读官方[认证](https://kubernetes.io/docs/admin/authentication/)和[授权](https://kubernetes.io/docs/admin/authorization/)文档。

## 集成

目前只支持`Heapster`集成，但是有计划将集成框架引入`Dashboard`。它将允许支持和集成更多指标采集器以及其他应用程序，如`[Weave Scope](https://github.com/weaveworks/scope)`或`[Grafana](https://github.com/grafana/grafana)`等。

### 指标集成

指标集成允许`Dashboard`显示cpu/内存使用情况图以及在集群内运行的资源的迷你图。为了使`Dashboard`能够适应度量标准提供程序的崩溃，引入了`--metric-client-check-period`标志。默认情况下，将检查度量标准提供程序每30秒的运行状况，如果崩溃，则将禁用度量标准。

#### Heapster

要在`Dashboard`中显示迷你图和使用情况图，需要在群集上运行`Heapster`。我们要求将`heapster`与名为`heapster`的`Service`一起部署在`kube-system`命名空间中。如果无法从群集内部访问`heapster`，则可以提供`heapster url`作为`Dashboard`容器的标志`--heapster-host=<heapster_url>`。

> 注意：目前`--heapster-host`标志不支持HTTPS连接。只应使用HTTP网址。

## FQA

1. 如何在开发环境使用HTTPS？

    开发环境以`gulp serve`或`gulp serve：prod`开始。要使其在HTTPS上运行，需要传递证书。为此，需要对所提到的命令使用以下标志：

    - tlsCert：`.crt`文件的路径
    - tlsKey：`.key`文件的路径
    - serveHttps：启动HTTPS模式的标识

2. Dashboard显示`open /certs/dashboard.crt: no such file or directoty`错误。

    **它已在1.8.0版本中修复，不应该再发生了。**

    > 这种情况时有发生。用于创建自签名证书的`Init`容器尚未完成其工作，并且已启动主`Dashboard`容器而没有所需的证书。尝试重新部署`Dashboard`，它应该解决问题。

3. 在`Dashboard`中无法看到图片，如何启动它？

    确保`Heapster`已启动并正在运行，`Dashboard`能够与之连接。首先，必须使用`Dashboard`或`kubectl`命令验证`Heapster`状态。然后，应该检查`Dashboard`日志并查找度量标准和`Heapster`关键字。 可以在[此处](/kubernetes/Dashboard/05-Integrations.md)找到有关`Dashboard`集成的更多信息。

4. 在开发过程中，在浏览器的控制台中收到了很多奇怪的错误。可能有什么不对？

    可能需要更新npm依赖项。从`Dashboard`的根目录运行以下命令：`rm -rf node_modules && npm i`

5. 为何显示`Go is not in the path`？

    遇到这样的错误可能意味着需要重新运行以下命令：`export PATH=$PATH:/usr/local/go/bin`

6. 如下报错：`linux mounts: Path /var/lib/kubelet is mounted on / but it is not a shared mount`

    尝试执行如下命令：`sudo mount --bind /var/lib/kubelet /var/lib/kubelet && sudo mount --make-shared /var/lib/kubelet`

    在[这里](https://github.com/kubernetes/kubernetes/issues/4869#issuecomment-193640483)获取更多信息。

7. 尝试访问`Dashboard`时看到404错误。无法加载仪表板资源。

    ```bash
    GET https://<IP>/api/v1/namespaces/kube-system/services/kubernetes-dashboard/static/vendor.9aa0b786.css
    proxy:1 GET https://<IP>/api/v1/namespaces/kube-system/services/kubernetes-dashboard/static/app.8ebf2901.css
    proxy:5 GET https://<IP>/api/v1/namespaces/kube-system/services/kubernetes-dashboard/api/appConfig.json
    proxy:5 GET https://<IP>/api/v1/namespaces/kube-system/services/kubernetes-dashboard/static/app.68d2caa2.js
    proxy:5 GET https://<IP>/api/v1/namespaces/kube-system/services/kubernetes-dashboard/static/vendor.840e639c.js
    proxy:5 GET https://<IP>/api/v1/namespaces/kube-system/services/kubernetes-dashboard/api/appConfig.json
    proxy:5 GET https://<IP>/api/v1/namespaces/kube-system/services/kubernetes-dashboard/static/app.68d2caa2.js
    ```

    重要提示：有一个与`Kubernetes 1.7.6`相关的[已知问题](https://github.com/kubernetes/kubernetes/issues/52729)，其中`/ui`重定向不起作用。尝试在`/ui`重定向`url`的末尾添加尾部斜杠：`http://localhost:8001/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy/`

    如果这没有帮助，那么这意味着集群存在问题，或者尝试以错误的方式访问`Dashboard`。通常，当尝试以错误的方式使用`kubectl proxy公开`Dashboard`时（即缺少权限），会发生这种情况。

    您可以快速检查是否访问 `http://localhost:8001/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard/` 而不是 `http://localhost:8001/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy`。

    检查问题是否与`Dashboard`相关的其他方法是使用“[访问`Dashboard`](/kubernetes/Dashboard/03-Accessing-Dashboard.md)”指南中描述的`NodePort`方法公开和访问它。这将允许直接访问`Dashboard`而不涉及任何代理。

    如果任何描述的方法都有效，那么这意味着它不是`Dashboard`问题，应该在核心存储库上寻求帮助，或者更好地阅读[Kubernetes文档](https://kubernetes.io/docs/tasks/)以了解它的工作原理。

8. 使用Kubernetes GCE集群，但获取禁止的访问错误。

    默认情况下，GCE上的`Dashboard`安装的权限很小。这不是问题。 应该授予`kubernetes-dashboard` `Service Account`更多权限，以便能够访问群集资源。阅读[Kubernetes文档](https://kubernetes.io/docs/tasks/)以了解如何执行此操作。您还可以查看[＃2326](https://github.com/kubernetes/dashboard/issues/2326)和[＃2415（评论）](https://github.com/kubernetes/dashboard/issues/2415#issuecomment-348370032)了解更多详情。

9. `/ui`重定向不起作用或显示`Error: 'malformed HTTP response`

    基于部署和访问`Dashboard`（HTTPS或HTTP）的方式，存在不同的问题。

    - **通过HTTP访问Dashboard**：有一个与`Kubernetes 1.7.X`相关的已知问题，其中`/ui`重定向不起作用。尝试在`/ui`重定向`url`的末尾添加尾部斜杠：`http://localhost:8001/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy/`
    - **通过HTTPS访问Dashboard**： `/ui`重定向不适用于HTTPS的原因是它尚未在核心存储库中更新。查看[这里](https://github.com/kubernetes/kubernetes/pull/53046#discussion_r145338754)了解什么时候合并。可能在K8S 1.8.3+之前不可用。

## kubectl-proxy

在`localhost`和`Kubernetes API Server`之间创建一个服务端代理或者应用级别的网关。它还允许通过指定的`HTTP`路径提供静态内容。所有传入的数据都通过一个端口进入并转发到远程`kubernetes API Server`端口，不包含静态内容匹配的路径。

```bash
Usage:
  kubectl proxy [--port=PORT] [--www=static-dir] [--www-prefix=prefix] [--api-prefix=prefix] [options]

Use "kubectl options" for a list of global command-line options (applies to all commands).

The following options can be passed to any command:

      --alsologtostderr=false: log to standard error as well as files
      --as='': Username to impersonate for the operation
      --as-group=[]: Group to impersonate for the operation, this flag can be repeated to specify multiple groups.
      --cache-dir='/home/sugoi/.kube/http-cache': Default HTTP cache directory
      --certificate-authority='': Path to a cert file for the certificate authority
      --client-certificate='': Path to a client certificate file for TLS
      --client-key='': Path to a client key file for TLS
      --cluster='': The name of the kubeconfig cluster to use
      --context='': The name of the kubeconfig context to use
      --insecure-skip-tls-verify=false: If true, the server's certificate will not be checked for validity. This will
make your HTTPS connections insecure
      --kubeconfig='': Path to the kubeconfig file to use for CLI requests.
      --log-backtrace-at=:0: when logging hits line file:N, emit a stack trace
      --log-dir='': If non-empty, write log files in this directory
      --log-file='': If non-empty, use this log file
      --log-file-max-size=1800: Defines the maximum size a log file can grow to. Unit is megabytes. If the value is 0,
the maximum file size is unlimited.
      --log-flush-frequency=5s: Maximum number of seconds between log flushes
      --logtostderr=true: log to standard error instead of files
      --match-server-version=false: Require server version to match client version
  -n, --namespace='': If present, the namespace scope for this CLI request
      --password='': Password for basic authentication to the API server
      --profile='none': Name of profile to capture. One of (none|cpu|heap|goroutine|threadcreate|block|mutex)
      --profile-output='profile.pprof': Name of the file to write the profile to
      --request-timeout='0': The length of time to wait before giving up on a single server request. Non-zero values
should contain a corresponding time unit (e.g. 1s, 2m, 3h). A value of zero means don't timeout requests.
  -s, --server='': The address and port of the Kubernetes API server
      --skip-headers=false: If true, avoid header prefixes in the log messages
      --skip-log-headers=false: If true, avoid headers when opening log files
      --stderrthreshold=2: logs at or above this threshold go to stderr
      --token='': Bearer token for authentication to the API server
      --user='': The name of the kubeconfig user to use
      --username='': Username for basic authentication to the API server
  -v, --v=0: number for the log level verbosity
      --vmodule=: comma-separated list of pattern=N settings for file-filtered logging
```

### 例如

代理全部`kubernetes api`：

```bash
kubectl proxy --api-prefix=/
```

代理部分`kubernetes api`和一些静态文件：

```bash
kubectl proxy --www=/my/files --www-prefix=/static/ --api-prefix=/api/

curl localhost:8001/api/v1/pods
```

代理整个`kubernetes api`到另一个根目录：

```bash
kubectl proxy --api-prefix=/custom/

curl localhost:8001/custom/api/v1/pods
```

代理端口修改为8011，静态内容的路径为`./local/www/`:

```bash
kubectl proxy --port=8011 --www=./local/www/
```

在任意本地端口上运行`kubernetes apiserver`的代理。服务器的选定端口将输出到`stdout`：

```bash
kubectl proxy --port=0
```

运行`kubernetes apiserver`的代理，将`api`前缀更改为`k8s-api`
这使得例如，获取`pods`的`api`运行在`localhost:8001/k8s-api/v1/pods/`

```bash
kubectl proxy --api-prefix=/k8s-api
```

### options

| 短参数 | 长参数                                                         | 解释                                                                                                  |
| ------ | ---------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
|        | `--accept-hosts='^localhost$,^127\.0\.0\.1$,^\[::1\]$'`          | 代理接受的主机的正则表达式                                                                            |
|        | `--accept-paths='^.*'`                                           | 代理接受的路径的正则表达式                                                                            |
|        | `--address='127.0.0.1'`                                          | 要服务的IP地址                                                                                        |
|        | `--api-prefix='/'`                                               | 用于提供代理API的前缀                                                                                 |
|        | `--disable-filter=false`                                         | 如果为`true`，则在代理中禁用请求筛选。 这很危险，当与可访问端口一起使用时，可能会使容易受到`XSRF`攻击 |
|        | `--keepalive=0s`                                                 | keepalive指定活动网络连接的保持活动期。 设置为`0`以禁用keepalive                                      |
| -p     | `--port=8001`                                                    | 运行代理的端口。 设置为`0`以选择随机端口                                                              |
|        | `--reject-methods='^$'`                                          | 代理应拒绝的HTTP方法的正则表达式（例如`--reject-methods ='POST，PUT，PATCH'`）                        |
|        | `--reject-paths='^/api/.*/pods/.*/exec,^/api/.*/pods/.*/attach'` | 代理应拒绝的路径的正则表达式。此处指定的路径即使被`--accept-paths`接受也将被拒绝。                    |
| -u     | `--unix-socket=''`                                               | 运行代理的Unix套接字                                                                                  |
| -w     | --www=''                                                         | 在指定前缀下提供给定目录中的静态文件                                                                  |
| -P     | --www-prefix='/static/'                                          | 如果指定了静态文件目录，则在下面提供静态文件的前缀                                                    |
