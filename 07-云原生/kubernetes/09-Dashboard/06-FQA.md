---
title: 06-FQA
date: 2020-04-14T10:09:14.202627+08:00
draft: false
---

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
