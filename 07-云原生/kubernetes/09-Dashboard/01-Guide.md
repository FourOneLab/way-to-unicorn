---
title: 01-Guide
date: 2020-04-14T10:09:14.202627+08:00
draft: false
---

`Kubernetes Dashboard`是一个通用的基于`WebUI`的`kubernetes`集群管理工具。用户管理运行在集群中的应用并对这些运行的应用进行快速的故障排查，同时也可以用于管理集群本身。

## 入门指南

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

通过以下地址访问`Dashboard`：http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

## 创建一个认证令牌（RBAC）

要了解如何创建示例用户并登录，请按照[创建示例用户指南](/kubernetes/Dashboard/04-Creating-sample-user.md)进行操作。

注意：

- `Kubeconfig`身份验证方法不支持外部身份提供程序或基于证书的身份验证。
- `Dashboard`只能通过HTTPS访问。
- `Heapster`必须在集群中运行才能获取指标并以提醒方式展示。在“[集成](/kubernetes/Dashboard/05-Integrations.md)”指南中阅读更多相关信息。

