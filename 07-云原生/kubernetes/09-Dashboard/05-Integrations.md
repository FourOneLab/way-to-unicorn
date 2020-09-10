---
title: 05-Integrations
date: 2020-04-14T10:09:14.202627+08:00
draft: false
---

目前只支持`Heapster`集成，但是有计划将集成框架引入`Dashboard`。它将允许支持和集成更多指标采集器以及其他应用程序，如`[Weave Scope](https://github.com/weaveworks/scope)`或`[Grafana](https://github.com/grafana/grafana)`等。

## 指标集成

指标集成允许`Dashboard`显示cpu/内存使用情况图以及在集群内运行的资源的迷你图。为了使`Dashboard`能够适应度量标准提供程序的崩溃，引入了`--metric-client-check-period`标志。默认情况下，将检查度量标准提供程序每30秒的运行状况，如果崩溃，则将禁用度量标准。

### Heapster

要在`Dashboard`中显示迷你图和使用情况图，需要在群集上运行`Heapster`。我们要求将`heapster`与名为`heapster`的`Service`一起部署在`kube-system`命名空间中。如果无法从群集内部访问`heapster`，则可以提供`heapster url`作为`Dashboard`容器的标志`--heapster-host=<heapster_url>`。

> 注意：目前`--heapster-host`标志不支持HTTPS连接。只应使用HTTP网址。
