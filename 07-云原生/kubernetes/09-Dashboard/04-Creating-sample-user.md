---
title: 04-Creating-sample-user
date: 2020-04-14T10:09:14.202627+08:00
draft: false
---

在本指南中，将了解如何使用`Kubernetes`的`Service Account`机制创建新用户，授予此用户管理员权限并使用与此用户关联的`Bearer Token`登录仪表板。

将下面提供的的代码片段复制到某个`dashboard-adminuser.yaml`文件，并使用`kubectl apply -f dashboard-adminuser.yaml`创建它们。

## 创建 Service Account

首先在`kube-system`命名空间中创建名为`admin-user`的`Service Account`。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```

## 创建 ClusterRoleBinding

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

## Bearer Token

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

