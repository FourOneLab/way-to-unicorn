---
title: kubectl-proxy
date: 2020-04-14T10:09:14.202627+08:00
draft: false
---

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

## 例如

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

## options

| 短参数 | 长参数                                                           | 解释                                                                                                  |
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
