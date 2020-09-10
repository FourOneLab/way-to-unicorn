---
title: 02-Access-Control
date: 2020-04-14T10:09:14.202627+08:00
draft: false
---
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

- `Authorization: Bearer <token> `：在每次向`Dashboard`发出请求时传递请求头。从`1.6`版开始支持。具有最高优先级。如果存在，则不会显示登录界面。
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
