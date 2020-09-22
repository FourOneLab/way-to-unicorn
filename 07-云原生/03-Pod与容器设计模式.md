# Pod与容器设计模式

## Pod

Pod 是Kubernetes项目中最小的API对象，原子调度单位。

> Docker的原理与本质：Namespace做隔离，Cgroups做限制，rootfs做文件系统。

**容器的本质：进程**。

- **容器**，是未来云计算系统中的进程
- **容器镜像**，是未来云计算系统里的“.exe”安装包
- **kubernetes**，就是这个云计算系统（操作系统）

### 进程与进程组

安装psmics工具集，并使用pstree工具查看系统进程。

```bash
yum install -y psmisc

# 返回当前系统中正在运行的进程的树状结构
pstree -g
systemd(1)───rsyslogd(627) ─┬─{rs:main Q:Reg}(627)
                       └─{in:imklog}(627)
# 省略很多
...

```

在一个正在运行的操作系统中，进程是以进程组的方式“有原则的”组织在一起，在进程后的括号中的数组表示进程组ID（Process Group ID，**PGID**）。

rsyslogd程序是负责Linux系统里的日志处理，它的主程序是main，和它要用的内核日志模块imklog等，同属于627进程组，这些进程相互协作，共同完成rsyslog程序的职责。**对于操作系统来说，这样的进程更容易管理**。例如，Linux系统只需要将信号（如SIGKILL）发送给进程组，那么该进程组中的所有进程都会收到这个信号而终止运行。

**kubernetes项目所做的，就是将“进程组”的概念映射到了容器技术中**，并使其成为云计算“操作系统”里面的“**一等公民**”。在实际应用中，应用之间有着密切的协作关系，类似于“进程与进程组”的关系，这使得它们必须部署在同一台机器上，否则基于Socket的通信和文件交换，会出现问题。没有组的概念，这样的运维关系非常难以处理。

**容器是单进程模型**，并不是指容器里只能运行“一个”进程，而是指**容器没有管理多个进程的能力**。这是因为容器里`PID=1`的进程就是应用本身，其他的进程都是这个`PID=1`进程的子进程。用户编写的应用，并不能够向正常操作系统里的init进程或者systemd那样拥有进程管理的功能。

> 举个例子：一个Java Web程序（PID=1），然后执行docker exec 进入该容器后，在后台启动一个Nginx进程（PID=3），当这个Nginx进程异常退出的时候如何知道？进程退出后的垃圾回收工作又该谁去做？

成组调度问题存在的问题：

1. Mesos采用资源囤积机制：在所有亲和性任务到达后才开始统一进行调度（囤积机制容易带来不可避免的调度效率损失和死锁的可能性）
2. Google Omega采用乐观调度机制：先不管冲突，通过精心设计的回滚机制在出现冲突后解决问题（客观调度的实现程度过于复杂）

在Kubernetes中以Pod为原子调度单位，调度器统一按照Pod而非容器的资源需求进行计算。

容器之间的紧密协作称为“超亲密关系”。这样的容器的典型特征包括但不限于：

1. 互相之间会发生**直接的文件交换**
2. 使用localhost或者Socket文件进行**本地通信**
3. 发生**非常频繁**的远程调用
4. 需要**共享某些 Linux Namespace**（比如，一个容器要加入另一个容器的 Network Namespace）
5. 。。。

容器的超亲密关系可以在调度层面实现，**Pod在kubernetes项目中，最重要的是“容器设计模式”**。

### Pod实现原理

Pod是**逻辑**上的概念。即Kubernetes真正处理的其实是宿主机操作系统上Linux容器的**Namespace**和**Cgroups**，而不存在一个所谓的Pod隔离环境。**Pod是一组共享了某些资源的容器**，Pod里的所有容器共享的是同一个**Network Namespace**，并且可以声明共享同一个**Volume**。

使用Docker原理能否实现Pod？A与B两个容器共享网络和Volume，如下命令：

```bash
docker run --net=B --volumes-from=B --name=A image-A ...

# 容器B必须比容器A先启动
```

这样的话，多个容器就**不是对等关系**，而是**拓扑关系**。

Pod的实现需要使用一个中间容器，这个容器叫作Infra容器。在这个Pod中，Infra容器永远都是**第一个**被创建的容器，而其他用户定义的容器，则通过Join Network Namespace的方式与Infra容器关联在一起。

![Infra Container](/images/infra-container.png)

这个Pod里有两个用户容器A和B，还有一个Infra容器。在Kubernetes项目中，Infra容器占用极少资源，使用一个非常特殊的镜像（`k8s.gcr.io/pause`）。**这个镜像是一个用汇编语言编写的，永远处于“暂停”状态的容器，解压后的大小也只有100~20KB左右**。

对于Pod里的容器A和B：

- 它们可以直接使用localhost通信
- 它们看到的网络设备与Infra容器看到的完全一样
- 一个Pod只有一个IP地址，是这个Pod的Network Namespace对应的IP地址
- 其他的所有网络资源，都是一个Pod一份，并且被该Pod中的所有容器共享
- **Pod的生命周期只跟Infra容器一致**，与容器A和B无关

对于同一个Pod里面的所有用户容器来说，它们的进出流量，可以认为都是通过Infra容器完成的。**如果要为 Kubernetes 开发一个网络插件，应该重点考虑的是如何配置这个 Pod 的 Network Namespace，而不是每一个用户容器如何使用网络配置，这是没有意义的**。

- 这意味着，如果网络插件需要在容器里安装某些包或者配置才能完成的话，是不可取的，**在Infra 容器镜像的 rootfs里几乎什么都没有**，没有你随意发挥的空间
- 这意味着，网络插件完全不必关心用户容器启动与否，而只需要关注如何配置 Pod，也就是 Infra 容器的Network Namespace 即可

有了这个设计之后，共享Volume就简单多了：Kubernetes项目只要把所有Volume的定义都设计在 Pod 层级即可。这样，一个Volume对应的宿主机目录对于Pod来说就只有一个，Pod里的容器只要声明挂载这个 Volume，就一定可以共享这个Volume对应的宿主机目录。比如下面这个例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
    hostPath:
      path: /data
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```

在这个例子中，debian-container 和 nginx-container 都声明挂载了 shared-data 这个Volume。而 shared-data 是 hostPath 类型。所以，它对应在宿主机上的目录就是：`/data`。而这个目录，其实就被同时绑定挂载进了上述两个容器当中。

### 容器设计模式

> 容器设计模式：启动一个辅助容器，来完成一些独立于主进程（主容器）之外的工作，他们共享Network Namespace，这样与Pod网络相关的配置和管理，交给sidecar完成。

Sidecar模式的应用场景：

- 应用与日志收集
- 代理容器
- 适配器容器

Pod 这种“超亲密关系”容器的设计思想，实际上就是希望，当用户想在一个容器里跑多个功能并不相关的应用时，应该优先考虑它们是不是更应该被描述成一个 Pod 里的多个容器。为了能够掌握这种思考方式，应该尽量尝试使用它来描述一些用单个容器难以解决的问题。

第一个最典型的例子是：**WAR 包与 Web 服务器**。我们现在有一个 Java Web 应用的 WAR 包，它需要被放在 Tomcat 的 webapps 目录下运行起来。假如，你现在只能用 Docker 来做这件事情，那该如何处理这个组合关系呢？

1. 一种方法是，把 WAR 包直接放在 Tomcat 镜像的 webapps 目录下，做成一个新的镜像运行起来。可是，这时候，如果你要更新WAR包的内容，或者要升级Tomcat镜像，就要重新制作一个新的发布镜像，非常麻烦
2. 另一种方法是，你压根儿不管 WAR 包，永远只发布一个 Tomcat 容器。不过，这个容器的webapps 目录，就必须声明一个 hostPath 类型的 Volume，从而把宿主机上的 WAR 包挂载进Tomcat 容器当中运行起来。不过，这样你就必须要解决一个问题，即：如何让每一台宿主机，都预先准备好这个存储有 WAR 包的目录呢？这样来看，你只能独立维护一套分布式存储系统了

实际上，有了 Pod 之后，这样的问题就很容易解决了。我们可以把 WAR 包和 Tomcat 分别做成镜像，然后把它们作为一个 Pod 里的两个容器“组合”在一起。这个 Pod 的配置文件如下所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: sample:v2
    name: war
    command: ["cp", "/sample.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: tomcat:7.0
    name: tomcat
    command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001
  volumes:
  - name: app-volume
    emptyDir: {}
```

在这个 Pod 中，我们定义了两个容器：

- 第一个容器使用的镜像是 `sample:v2`，这个镜像里只有一个 WAR 包（`sample.war`）放在根目录下
- 第二个容器则使用的是一个标准的Tomcat镜像

WAR 包容器的类型不再是一个普通容器，而是一个Init Container类型的容器，在 Pod 中，所有Init Container定义的容器，都会比 `spec.containers` 定义的用户容器**先启动**。并且，Init Container 容器会**按顺序逐一启动**，而直到它们都启动并且退出了，**用户容器才会启动**。

1. 这个 Init Container 类型的 WAR 包容器启动后，执行了一句 "`cp /sample.war /app`"，把应用的 WAR 包拷贝到 `/app` 目录下，然后退出
2. 而后这个 `/app` 目录，就挂载了一个名叫 app-volume 的 Volume
3. 接下来就很关键了。Tomcat 容器，同样声明了挂载 app-volume 到自己的 webapps 目录下。所以，等 Tomcat 容器启动时，它的 webapps 目录下就一定会存在 sample.war 文件：这个文件正是 WAR 包容器启动时拷贝到这个 Volume 里面的，而这个 Volume 是被这两个容器共享的

像这样，用一种“组合”方式，解决了 WAR 包与 Tomcat 容器之间耦合关系的问题。实际上，**这个所谓的“组合”操作，正是容器设计模式里最常用的一种模式，它的名字叫：sidecar。**

> sidecar 指的就是可以在一个 Pod 中，启动一个辅助容器，来完成一些独立于主进程（主容器）之外的工作。

比如，这个应用Pod中，Tomcat容器是要使用的**主容器**，而WAR包容器的存在，只是为了给它提供一个WAR包而已。所以，用InitContainer的方式优先运行WAR包容器，扮演了一个 sidecar 的角色。

第二个例子：**容器的日志收集**。比如，现在有一个应用，需要不断地把日志文件输出到容器的 `/var/log` 目录中。

1. 把一个 Pod 里的 Volume 挂载到应用容器的 `/var/log` 目录上
2. 在这个 Pod 里同时运行一个 sidecar 容器，它也声明挂载同一个 Volume 到自己的 `/var/log` 目录上
3. sidecar容器就只需要做一件事儿，不断地从自己的`/var/log`目录里读取日志文件，转发到 MongoDB 或者 Elasticsearch 中存储起来，一个最基本的日志收集工作就完成了

这个例子中的 sidecar 的主要工作也是使用共享的 Volume 来完成对文件的操作。

Pod中所有容器都共享同一个Network Namespace。这就使得很多与 Pod 网络相关的配置和管理，也都可以交给 sidecar 完成，而完全无须干涉用户容器。最典型的例子莫过于**Istio**这个微服务治理项目使用sidecar容器完成微服务治理。

容器技术的本质是“**进程**”，一个运行在虚拟机里的应用，是被管理在Systemd或者supervisord执行的**一组进程，而不是一个进程。**Pod实际上在扮演传统基础设施里的**虚拟机**的角色，而容器，则是这个虚拟机里运行的用户程序。

当需要把一个运行在虚拟机里的应用迁移到 Docker 容器中时，一定要仔细分析到底有哪些进程（组件）运行在这个虚拟机里。然后，就可以**把整个虚拟机想象成为一个Pod**，把这些进程分别做成容器镜像，把有顺序关系的容器，定义为 Init Container。这才是更加合理的、松耦合的容器编排诀窍，也是从传统应用架构，到“微服务架构”最自然的过渡方式。**如果强行把整个应用塞到一个容器里，甚至不惜使用Docker in Docker 这种在生产环境中后患无穷的解决方案，恐怕最后往往得不偿失**。

### Pod的定义

Pod，而不是容器，才是 Kubernetes 项目中的最小编排单位。将这个设计落实到 API 对象上，容器（Container）就成了 Pod 属性里的一个普通的字段。那么问题来了：到底哪些属性属于 Pod 对象，而又有哪些属性属于 Container 呢？

> **Pod扮演的是传统部署环境里“虚拟机”的角色**。这样的设计，是为了使用户从传统环境（虚拟机环境）向 Kubernetes（容器环境）的迁移，更加平滑。

如果把 **Pod** 看成传统环境里的“**机器**”、把**容器**看作是运行在这个“机器”里的“**用户程序**”，那么很多关于 Pod 对象的设计就非常容易理解了。比如，凡是**调度**、**网络**、**存储**，以及**安全**相关的属性，基本上是 **Pod 级别的**。这些属性的共同特征是，它们描述的是“机器”这个整体，而不是里面运行的“程序”。比如:

1. 配置这个“机器”的网卡（即：Pod 的网络定义）
2. 配置这个“机器”的磁盘（即：Pod 的存储定义）
3. 配置这个“机器”的防火墙（即：Pod 的安全定义）
4. 这台“机器”运行在哪个服务器之上（即：Pod 的调度）

NodeSelector字段，是一个供用户将 Pod 与 Node 进行绑定的字段，用法如下所示：

``` yaml
apiVersion: v1
kind: Pod
...
spec:
 nodeSelector:
 disktype: ssd
```

这样的一个配置，意味着这个 Pod 永远只能运行在携带了“disktype:ssd”标签（label）的节点上；否则，它将调度失败。

NodeName字段，一旦 Pod 的这个字段被赋值，Kubernetes项目就会被认为这个Pod已经经过了调度，**调度的结果就是赋值的节点名字**。所以，**这个字段一般由调度器负责设置**，但用户也可以设置它来“骗过”调度器，当然这个做法一般是在**测试**或者**调试**的时候才会用到。

HostAliases字段，定义了 Pod 的 hosts 文件（比如`/etc/hosts`）里的内容，用法如下：

``` YAML
apiVersion: v1
kind: Pod
...
spec:
  hostAliases:
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
...
```

在这个 Pod 的 YAML 文件中，设置了一组 IP 和 hostname 的数据。这样，这个 Pod 启动后，`/etc/hosts`文件的内容将如下所示：

``` bash
cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1 localhost
...
10.244.135.10 hostaliases-pod
10.1.2.3 foo.remote
10.1.2.3 bar.remote
```

其中，最下面两行记录，就是通过 HostAliases 字段为 Pod 设置的。需要指出的是：

- 在Kubernetes 项目中，如果要设置hosts文件里的内容，一定要通过这种方法
- 如果直接修改了 hosts 文件的话，在 Pod 被删除重建之后，kubelet会自动覆盖掉被修改的内容

除了上述跟“机器”相关的配置外，凡是跟容器的 Linux Namespace 相关的属性，也一定是 Pod级别的。Pod的设计，就是要让它里面的容器尽可能多地共享 Linux Namespace，仅保留必要的隔离和限制能力。这样，Pod模拟出的效果，就跟虚拟机里程序间的关系非常类似了。

举个例子，在下面这个 Pod 的 YAML 文件中，定义 `shareProcessNamespace=true`：

``` YAML
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```

这就意味着这个 Pod 里的容器要共享 PID Namespace。而在这个 YAML 文件中，还定义了两个容器：一个是 nginx 容器，一个是开启了 tty 和 stdin 的shell容器。在 Pod 的 YAML 文件里声明开启它们俩，其实等同于设置了 docker run 里的 -it（-i 即 stdin，-t 即 tty）参数。这个 Pod 被创建后，就可以使用shell容器的tty跟这个容器进行交互了。

> 可以直接认为 **tty** 就是Linux给用户提供的一个常驻小程序，**用于接收用户的标准输入，返回操作系统的标准输出。**当然，为了能够在 tty 中输入信息，你还需要同时开启 stdin（标准输入流）。

``` bash
# 创建这个共享PID Namespace的Pod
kubectl create -f nginx.yaml

# 使用kubectl attach 命令，连接到shell容器的tty上
kubectl attach -it nginx -c shell

# ps ax
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    8 root      0:00 nginx: master process nginx -g daemon off;
   14 101       0:00 nginx: worker process
   15 root      0:00 sh
   21 root      0:00 ps ax
```

在这个容器里，我们不仅可以看到它本身的 `ps ax` 指令，还可以看到 **nginx 容器的进程**，以及 **Infra容器的 /pause 进程**。这就意味着，整个 Pod 里的每个容器的进程，对于所有容器来说都是可见的：**它们共享了同一个 PID Namespace**。类似地，**凡是 Pod 中的容器要共享宿主机的 Namespace，也一定是Pod级别的定义**，比如:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  hostNetwork: true
  hostIPC: true
  hostPID: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```

在这个 Pod 中，定义了共享宿主机的 **Network**、**IPC** 和 **PID** Namespace。这就意味着，这个Pod 里的所有容器:

1. 会直接使用宿主机的网络
2. 直接与宿主机进行IPC通信
3. 看到宿主机里正在运行的所有进程

当然，除了这些属性，Pod里最重要的字段当属“**Containers**”了。

> "container"与“Init Containers”。其实，这两个字段都属于Pod对容器的定义，内容也完全相同，只是 Init Containers 的生命周期，会先于所有的Containers，并且严格按照定义的顺序执行。

Kubernetes 项目中对 Container 的定义，和 Docker 相比并没有什么太大区别。容器技术概念中：

- Image（镜像）
- Command（启动命令）
- workingDir（容器的工作目录）
- Ports（容器要开放的端口）
- volume Mounts（容器要挂载的 Volume）

都是构成 Kubernetes 项目中 Container 的主要字段。不过在这里，还有几个属性值得额外关注。

1. 首先，是 **ImagePullPolicy** 字段。它**定义了镜像拉取的策略**。而它之所以是一个 Container 级别的属性，是因为容器镜像本来就是 Container 定义中的一部分
   1. ImagePullPolicy 的值默认是**Always**，即每次创建Pod都**重新拉取**一次镜像。另外，当容器的镜像是类似于 nginx 或者 nginx:latest 这样的名字时，ImagePullPolicy 也会被认为 Always
   2. 而如果它的值被定义为 **Never** 或者 **IfNotPresent**，则意味着 Pod 永远不会主动拉取这个镜像，或者只在宿主机上不存在这个镜像时才拉取
2. 其次，是 **Lifecycle** 字段。它定义的是 Container Lifecycle Hooks。顾名思义，Container Lifecycle Hooks 的作用，是在容器状态发生变化时触发一系列“钩子”。

我们来看这样一个例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
```

这是一个来自 Kubernetes 官方文档的 Pod 的 YAML 文件。它其实非常简单，只是定义了一个nginx镜像的容器。不过，在这个 YAML 文件的容器（Containers）部分，你会看到这个容器分别设置了一个 **postStart** 和 **preStop** 参数。

- postStart：指的是在容器启动后，立刻执行一个指定的操作。需要明确的是，postStart 定义的操作，**虽然是在 Docker 容器 ENTRYPOINT 执行之后，但它并不严格保证顺序**。也就是说，在 postStart 启动时，ENTRYPOINT 有可能还没有结束。当然，如果 postStart 执行超时或者错误，Kubernetes 会在该 Pod 的 Events 中报出该容器启动失败的错误信息，导致 Pod 也处于失败的状态。
- preStop：指的是在容器被杀死之前（比如，收到了 SIGKILL 信号）。preStop操作的执行，**是同步的**。所以，它会阻塞当前的容器杀死流程，直到这个Hook定义操作完成之后，才允许容器被杀死，**这跟 postStart 不一样**。

所以，在这个例子中，我们在容器成功启动之后，在`/usr/share/message`里写入了一句“欢迎信息”（即postStart定义的操作）。而在这个容器被删除之前，我们则先调用了 nginx 的退出指令（即 preStop 定义的操作），从而实现了容器的“优雅退出”。

### Pod 生命周期

Pod 对象在 Kubernetes 中的生命周期。Pod 生命周期的变化，主要体现在 Pod API 对象的**Status** 部分，这是它除了 Metadata 和 Spec 之外的**第三个重要字段**。其中，`pod.status.phase`，就是 Pod的当前状态，它有如下几种可能的情况：

1. Pending：Pod的YAML文件已经提交给了Kubernetes，API对象已经被创建并保存在 Etcd 当中。但是，这个 Pod 里有些容器因为某种原因而不能被顺利创建，比如，调度不成功
2. Running：Pod已经调度成功，跟一个具体的节点绑定。它包含的容器都已经创建成功，并且**至少有一个正在运行中**。
3. Succeeded：Pod里的所有容器都正常运行完毕，并且已经退出了，这种情况在运行一次性任务时最为常见。
4. Failed：Pod里至少有一个容器以不正常的状态（非 0 的返回码）**退出**。这个状态的出现，意味着得想办法 Debug 这个容器的应用，比如查看 Pod 的 Events 和日志。
5. Unknown：异常状态，意味着 Pod 的状态不能持续地被 kubelet 汇报给 kubeapiserver，**这很有可能是主从节点（Master和 Kubelet）间的通信出现了问题**。

Pod 对象的 Status 字段，还可以再细分出一组 Conditions。这些细分状态的值包括：

- PodScheduled
- Ready
- Initialized
- Unschedulable

它们主要用于描述造成当前Status的具体原因是什么。比如，Pod 当前的 Status 是 Pending，对应的 Condition 是Unschedulable，这就意味着它的调度出现了问题。而其中，Ready 这个细分状态非常值得我们关注：它意味着 Pod 不仅已经正常启动（Running 状态），而且已经可以对外提供服务了。**Running和Ready是有区别的**。

Pod 的这些状态信息，是我们判断应用运行情况的重要标准，尤其是 Pod 进入了非“Running”状态后，一定要能迅速做出反应，根据它所代表的异常情况开始跟踪和定位，而不是去手忙脚乱地查阅文档。

对于 Pod 状态是 Ready，实际上不能提供服务的情况能想到几个例子：

1. 程序本身有bug，本来应该返回 200，但因为代码问题，返回的是500
2. 程序因为内存问题，已经僵死，但进程还在，但无响应
3. Dockerfile写的不规范，应用程序不是主进程，那么主进程出了什么问题都无法发现
4. 程序出现死循环

### Pod的特殊字段

Pod中特殊的Volume：Project  Volume（投射数据卷，是Kubernetes v1.11之后的新特性）。在 Kubernetes 中，有几种特殊的 Volume，它们存在的意义：

- 不是为了存放容器里的数据
- 也不是用来进行容器和宿主机之间的数据交换

这些特殊 Volume 的作用，是**为容器提供预先定义好的数据**。所以，从容器的角度来看，这些 Volume 里的信息仿佛是被 Kubernetes“投射”（Project）进入容器当中的。这正是 Projected Volume 的含义。到目前为止，Kubernetes 支持的 Projected Volume 一共有四种：

1. Secret
2. ConfigMap
3. Downward API
4. ServiceAccountToken

#### Secret

作用是把 Pod 想要访问的**加密数据**，存放到 Etcd 中。然后，就可以通过在 Pod 的容器里挂载 Volume 的方式，访问到这些 Secret里保存的信息了。Secret 最典型的使用场景，莫过于存放数据库的 Credential 信息，比如下面这个例子：

``` YAML
apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume
spec:
  containers:
  - name: test-secret-volume
    image: busybox
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: mysql-cred
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: mysql-cred
    projected:
      sources:
      - secret:
          name: user
      - secret:
          name: pass
```

在这个 Pod 中，定义了一个简单的容器。它声明挂载的Volume，并不是常见的emptyDir或者hostPath类型，而是 **projected** 类型。而这个Volume的数据来源（**sources**），则是名为user和pass的Secret对象，分别对应的是数据库的用户名和密码。这里用到的数据库的用户名、密码，正是以 Secret 对象的方式交给 Kubernetes 保存的。完成这个操作的指令，如下所示：

```bash
# username.txt 和 password.txt 文件里，存放的就是用户名和密码
cat ./username.txt
admin
cat ./password.txt
c1oudc0w!

# user和pass是为 Secret 对象指定的名字
kubectl create secret generic user --from-file=./username.txt
kubectl create secret generic pass --from-file=./password.txt

# 查看Secret对象
kubectl get secrets
NAME           TYPE                                DATA      AGE
user          Opaque                                1         51s
pass          Opaque                                1         51s
```

当然，除了使用 kubectl create secret 指令外，也可以直接通过编写 YAML 文件的方式来创建这个 Secret 对象，比如：

``` YAML
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  user: YWRtaW4=
  pass: MWYyZDFlMmU2N2Rm
```

可以看到，通过编写 YAML 文件创建出来的 Secret 对象只有一个。但它的 data 字段，却以 KeyValue的格式保存了两份Secret数据。其中：

- “user”是第一份数据的Key
- “pass”是第二份数据的Key

需要注意的是，Secret对象要求这些数据必须是经过**Base64转码**的，以免出现明文密码的安全隐患。这个转码操作也很简单，比如：

``` bash
echo -n 'admin' | base64
YWRtaW4=
echo -n '1f2d1e2e67df' | base64
MWYyZDFlMmU2N2Rm
```

这里需要注意的是，像这样创建的 Secret  对象，它里面的内容**仅仅是经过了转码，而并没有被加密**。在真正的生产环境中，需要在Kubernetes中**开启 Secret 的加密插件，增强数据的安全性**。

``` bash
# 创建Pod
kubectl create -f test-projected-volume.yaml

# 当 Pod 变成 Running 状态之后
# 再验证这些 Secret 对象是不是已经在容器里
kubectl exec -it test-projected-volume -- /bin/sh
ls /projected-volume/
user
pass

cat /projected-volume/user
root

cat /projected-volume/pass
1f2d1e2e67df
```

从返回结果中，可以看到，保存在 Etcd 里的用户名和密码信息，已经**以文件的形式出现在了容器的 Volume 目录里**。而这个文件的名字，就是 kubectl create secret 指定的 Key，或者说是Secret对象的 data 字段指定的 Key。**更重要的是，像这样通过挂载方式进入到容器里的 Secret，一旦其对应的 Etcd 里的数据被更新，这些 Volume 里的文件内容，同样也会被更新。**

其实，这是 kubelet 组件在定时维护这些Volume。需要注意的是，**这个更新可能会有一定的延时**。**所以在编写应用程序时，在发起数据库连接的代码处写好重试和超时的逻辑，绝对是个好习惯。**

Secret 类型种类比较多，下面列了常用的四种类型：

- 第一种是 Opaque，它是普通的 Secret 文件；
- 第二种是 service-account-token，是用于 service-account 身份认证用的 Secret；
- 第三种是 dockerconfigjson，这是拉取私有仓库镜像的用的一种 Secret；
- 第四种是 bootstrap.token，是用于节点接入集群校验用的 Secret。

#### ConfigMap

与 Secret 类似的是 ConfigMap，它与 Secret 的区别在于，ConfigMap 保存的是**不需要加密的、应用所需的配置信息**。而 ConfigMap 的用法几乎与 Secret 完全相同：

- 可以使用 kubectl create configmap 从文件或者目录创建 ConfigMap
- 也可以直接编写 ConfigMap 对象的 YAML 文件

比如，一个 Java 应用所需的配置文件（**.properties文件**），就可以通过下面这样的方式保存在ConfigMap里：

``` bash
# 查看 .properties 文件的内容
cat example/ui.properties
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice

# 从.properties 文件创建 ConfigMap
kubectl create configmap ui-config --from-file=example/ui.properties

# 查看这个 ConfigMap 里保存的信息 (data)
kubectl get configmaps ui-config -o yaml
apiVersion: v1
data:
  ui.properties:
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
  name: ui-config
  ...
```

> kubectl get -o yaml 这样的参数，会将指定的 Pod API 对象以 YAML 的方式展示出来。

现在对 ConfigMap 的使用做一个总结，以及它的一些注意点，注意点一共列了以下五条：

- 第一个注意点是ConfigMap 文件的大小。虽然说 ConfigMap 文件没有大小限制，但是在 ETCD 里面，数据的写入是有大小限制的，现在是限制在 1MB 以内；
- 第二个注意点是 pod 引入 ConfigMap 的时候，必须是相同的 Namespace 中的 ConfigMap，`ConfigMap.metadata` 里面是有 namespace 字段的；
- 第三个是 pod 引用的 ConfigMap。假如这个 ConfigMap 不存在，那么这个 pod 是无法创建成功的，其实这也表示在创建 pod 前，必须先把要引用的 ConfigMap 创建好；
- 第四点就是使用 envFrom 的方式。把 ConfigMap 里面所有的信息导入成环境变量时，如果 ConfigMap 里有些 key 是无效的，比如 key 的名字里面带有数字，那么这个环境变量其实是不会注入容器的，它会被忽略。但是这个 pod 本身是可以创建的。这个和第三点是不一样的方式，是 ConfigMap 文件存在基础上，整体导入成环境变量的一种形式；
- 最后一点是：只有通过 K8s api 创建的 pod 才能使用 ConfigMap，比如说通过用命令行 kubectl 来创建的 pod，肯定是可以使用 ConfigMap 的，但其他方式创建的 pod，比如说 kubelet 通过 manifest 创建的 static pod，它是不能使用 ConfigMap 的。

#### Downward API

它的作用是，让 Pod 里的容器能够直接获取到这个 Pod API 对象本身的信息。

``` YAML
apiVersion: v1
kind: Pod
metadata:
  name: test-downwardapi-volume
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
spec:
  containers:
    - name: client-container
      image: k8s.gcr.io/busybox
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
          readOnly: false
  volumes:
    - name: podinfo
      projected:
        sources:
        - downwardAPI:
            items:
              - path: "labels"
                fieldRef:
                  fieldPath: metadata.labels
```

在这个 Pod 的 YAML 文件中，定义了一个简单的容器，声明了一个 projected 类型的Volume。只不过这次 Volume 的数据来源，变成了 Downward API。而这个 Downward API Volume，则声明了要暴露 Pod 的 `metadata.labels` 信息给容器。

通过这样的声明方式，当前 Pod 的 Labels 字段的值，就会被 Kubernetes 自动挂载成为容器里的 `/etc/podinfo/labels` 文件。而这个容器的启动命令，则是不断打印出 `/etc/podinfo/labels` 里的内容。所以，当创建了这个Pod 之后，就可以通过 kubectl logs 指令，查看到这些 Labels 字段被打印出来，如下所示：

``` bash
kubectl create -f dapi-volume.yaml
kubectl logs test-downwardapi-volume
cluster="test-cluster1"
rack="rack-22"
zone="us-est-coast"
```

目前，Downward API 支持的字段已经非常丰富了，比如：

使用 fieldRef 可以声明使用:

字段名 | 描述
---|---
spec.nodeName | 宿主机名字
status.hostIP | 宿主机 IP
metadata.name| Pod 的名字
metadata.namespace | Pod 的 Namespace
status.podIP | Pod 的 IP
spec.serviceAccountName | Pod 的 Service Account 的名字
metadata.uid | Pod 的 UID
metadata.labels[`<KEY>`] |指定 `<KEY>` 的 Label 值
metadata.annotations[`<KEY>`] | 指定 `<KEY>` 的 Annotation 值
metadata.labels | Pod 的所有 Label
metadata.annotations | Pod 的所有 Annotation

使用 resourceFieldRef 可以声明使用:

- 容器的 CPU limit
- 容器的 CPU request
- 容器的 memory limit
- 容器的 memory request

> 上面这个列表的内容，仅供参考，在使用 Downward API 时，还是要去查阅一下官方文档。

需要注意的是，**Downward API 能够获取到的信息，一定是 Pod 里的容器进程启动之前就能够确定下来的信息。**如果想要获取 Pod 容器运行后才会出现的信息，比如，容器进程的PID，那就肯定不能使用 Downward API 了，而**应该考虑在 Pod 里定义一个 sidecar 容器**。

**Secret**、**ConfigMap**，以及 **Downward** **API** 这三种 Projected Volume 定义的信息，大多还可以通过**环境变量**的方式出现在容器里。但是，通过环境变量获取这些信息的方式，**不具备自动更新的能力**。所以，一般情况下，建议使用 Volume 文件的方式获取这些信息。

#### Service Account

Pod中与Secret密切相关的Service Account。

比如，现在有一个 Pod，在这个 Pod 里安装一个Kubernetes的Client，这样就可以从容器里直接访问并且操作这个 Kubernetes 的 API 了。不过，首先要解决 API Server 的**授权问题**。Service Account 对象的作用，就是 Kubernetes 系统内置的一种“**服务账户**”，它是 Kubernetes 进行**权限分配的对象**。比如：Service Account A，可以只被允许对 Kubernetes API 进行 GET 操作，Service Account B，则可以有 Kubernetes API 的所有操作的权限。

1. 像这样的 Service Account 的授权信息和文件，实际上保存在它所绑定的一个特殊的 Secret 对象里的。这个特殊的 Secret 对象，就叫作**ServiceAccountToken**
2. 任何运行在 Kubernetes 集群上的应用，都必须使用这个 **ServiceAccountToken** 里保存的授权信息，也就是**Token**，才可以合法地访问 API Server
3. 所以说，Kubernetes 项目的 Projected Volume 其实只有三种，因为第四种ServiceAccountToken，只是一种特殊的 Secret 而已

为了方便使用，Kubernetes 提供了一个的默认“服务账户”（**default Service Account**）。并且，**任何一个运行在 Kubernetes 里的 Pod，都可以直接使用这个默认的 Service Account，而无需显示地声明挂载它（使用Projected Volume机制实现）。**

``` bash
# 查看任意一个运行在 Kubernetes 集群里的 Pod
# 都已经自动声明一个类型是 Secret、名为 default-token-xxxx 的 Volume
# 自动挂载在每个容器的一个固定目录上

kubectl describe pod nginx-deployment-5c678cfb6d-lg9lw
Containers:
...
  Mounts:
    /var/run/secrets/kubernetes.io/serviceaccount from default-token-s8rbq (ro)
Volumes:
  default-token-s8rbq:
  Type:       Secret (a volume populated by a Secret)
  SecretName:  default-token-s8rbq
  Optional:    false
```

这个 Secret 类型的 Volume，正是默认 Service Account 对应的 ServiceAccountToken。Kubernetes在每个Pod创建的时候，自动在它的`spec.volumes`部分添加上了默认ServiceAccountToken的定义，然后自动给每个容器加上了对应的 volumeMounts 字段，**这个过程对于用户来说是完全透明的**。这样，一旦Pod创建完成，容器里的应用就可以直接从这个默认 ServiceAccountToken 的挂载目录里访问到授权信息和文件。

这个容器内的路径在 Kubernetes 里是固定的，即`/var/run/secrets/kubernetes.io/serviceaccount`。而这个 Secret 类型的Volume里面的内容如下所示：

```bash
ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt namespace  token
```

1. 应用程序只要直接加载这些授权文件，就可以访问并操作 Kubernetes API 了
2. 如果使用的是 Kubernetes 官方的 Client 包（`k8s.io/client-go`）的话，它还可以自动加载这个目录下的文件，不需要做任何配置或者编码操作

这种把 Kubernetes客户端以容器的方式运行在集群里，然后使用 default Service Account 自动授权的方式，被称作“**InClusterConfig**”，也是最推荐的进行 Kubernetes API 编程的授权方式。考虑到自动挂载默认 ServiceAccountToken 的**潜在风险**，Kubernetes 允许默认不为Pod 里的容器自动挂载这个 Volume。

- 除了这个默认的Service Account外，很多时候需要创建自己定义的 Service Account，来对应不同的权限设置
- Pod里的容器就可以通过挂载这些Service Account对应的ServiceAccountToken，来使用这些自定义的授权信息

#### SecurtyContext

SecurityContext 主要是用于限制容器的一个行为，它能保证系统和其他容器的安全。这一块的能力不是 Kubernetes 或者容器 runtime 本身的能力，而是 Kubernetes 和 runtime 通过用户的配置，最后下传到内核里，再通过内核的机制让 SecurityContext 来生效。

SecurityContext 主要分为三个级别：

- 第一个是容器级别，仅对容器生效；
- 第二个是 pod 级别，对 pod 里所有容器生效；
- 第三个是集群级别，就是 PSP，对集群内所有 pod 生效。

权限和访问控制设置项，现在一共列有七项（这个数量后续可能会变化）：

- 第一个就是通过用户 ID 和组 ID 来控制文件访问权限；
- 第二个是 SELinux，它是通过策略配置来控制用户或者进程对文件的访问控制；
- 第三个是特权容器；
- 第四个是 Capabilities，它也是给特定进程来配置一个 privileged 能力；
- 第五个是 AppArmor，它也是通过一些配置文件来控制可执行文件的一个访问控制权限，比如说一些端口的读写；
- 第六个是一个对系统调用的控制；
- 第七个是对子进程能否获取比父亲更多的权限的一个限制。

#### livenessProbe

在 Kubernetes 中，可以为 Pod 里的容器定义一个健康检查“探针”（Probe）。kubelet 会根据这个 Probe 的返回值决定这个容器的状态，而不是直接以容器进程是否运行（来自Docker返回的信息）作为依据。这种机制，是生产环境中保证应用健康存活的重要手段。

Kubernetes 文档中的例子：

``` YAML
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: test-liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

在这个 Pod 中，定义了一个有趣的容器。

1. 它在启动之后做的第一件事，就是在 `/tmp` 目录下创建了一个healthy文件，以此作为自己已经**正常运行**的标志
2. 而 30 s 过后，它会把这个文件删除掉
3. 与此同时，定义了一个livenessProbe，它的类型是exec，这意味着，它会在容器启动后，在容器里面执行一句指定的命令，比如：`cat /tmp/healthy`
4. 如果文件存在，这条命令的返回值就是 0，Pod 就会认为这个容器不仅已经启动，而且是健康的

这个健康检查，在容器启动5s后开始执行（`initialDelaySeconds:5`），每5s执行一次（`periodSeconds:5`）。30秒之后，查看Pod的Events，会报告容器是不健康的。再次查看Pod的状态，**这个异常的容器已经被Kubernetes重启了，在这个过程中，Pod报错Running状态不变**。

> Kubernetes中并没有Docker的Stop语义。所以虽然是Restart（重启），但实际却是**重新创建了容器**。

这个功能就是 Kubernetes 里的Pod恢复机制，也叫 **restartPolicy**。它是 Pod 的 Spec 部分的一个标准字段（`pod.spec.restartPolicy`），默认值是 **Always**，即：**任何时候这个容器发生了异常，它一定会被重新创建。**但一定要强调的是，**Pod 的恢复过程，永远都是发生在当前节点上，而不会跑到别的节点上去**。

事实上，一旦一个Pod与一个节点（Node）绑定，除非这个绑定发生了变化（`pod.spec.node` 字段被修改），否则它永远都不会离开这个节点。这也就意味着，**如果这个宿主机宕机了，这个 Pod 也不会主动迁移到其他节点上去。**

而如果想让Pod出现在其他的可用节点上，就必须使用 Deployment 这样的“控制器”来管理Pod，哪怕你只需要一个 Pod 副本。**这就是一个单 Pod 的 Deployment 与一个 Pod 最主要的区别。**

而作为用户，可以通过设置 restartPolicy，改变 Pod 的恢复策略:

- Always：在任何情况下，只要容器不在运行状态，就自动重启容器
- OnFailure：只在容器异常时才自动重启容器
- Never：从来不重启容器

**在实际使用时，需要根据应用运行的特性，合理设置这三种恢复策略。**

> 比如，一个 Pod，它只计算 1+1=2，计算完成输出结果后退出，变成 Succeeded 状态。这时，你如果再用 `restartPolicy=Always` 强制重启这个 Pod 的容器，就没有任何意义了。
>
> 而如果要关心这个容器退出后的上下文环境，比如容器退出后的日志、文件和目录，就需要将restartPolicy设置为Never。**因为一旦容器被自动重新创建，这些内容就有可能丢失掉了（被垃圾回收了）**。

值得一提的是，Kubernetes 的官方文档，把 restartPolicy 和 Pod 里容器的状态，以及 Pod 状态的对应关系，总结了非常复杂的一大堆情况。实际上，只要记住如下两个基本的设计原理即可：

1. 只要 Pod 的 restartPolicy 指定的策略**允许重启异常的容器**（比如：Always），那么这个Pod就会保持Running状态，并进行容器重启，否则，Pod 就会进入 Failed 状态
2. 对于包含多个容器的Pod，只有它里面**所有**的容器都进入异常状态后，Pod 才会进入 Failed 状态。在此之前，Pod都是Running状态。此时，Pod的READY字段会显示正常容器的个数，比如：

```bash
kubectl get pod test-liveness-exec
NAME           READY     STATUS    RESTARTS   AGE
liveness-exec   0/1       Running   1          1m
```

- 假如一个Pod里只有**一个容器**，然后这个容器异常退出了。那么，只有当restartPolicy=Never 时，这个Pod才会进入Failed状态。而其他情况下，Kubernetes 都可以重启这个容器，所以 Pod 的状态保持 Running 不变
- 而如果这个Pod有**多个容器**，仅有一个容器异常退出，它就始终保持 Running 状态，哪怕即使restartPolicy=Never，只有当所有容器也异常退出之后，这个 Pod 才会进入 Failed 状态

**其他情况，都可以以此类推出来。**除了在容器中**执行命令**外，livenessProbe也可以定义为**发起HTTP或者TCP请求**的方式，定义格式如下：

```yaml
...
livenessProbe:
     httpGet:
       path: /healthz
       port: 8080
       httpHeaders:
       - name: X-Custom-Header
         value: Awesome
       initialDelaySeconds: 3
       periodSeconds: 3
```

```yaml
...
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
```

Pod 可以暴露一个健康检查URL（比如`/healthz`），直接让健康检查去检测应用的监听端口。**这两种配置方法，在Web服务类的应用中非常常用**。

#### readinessProbe

readinessProbe 检查结果的成功与否，决定的这个Pod是不是能被通过Service的方式访问到，而并不影响 Pod 的生命周期。

#### PodPreset

Pod 的字段这么多，Kubernetes 能够自动给 Pod 填充某些字段。比如，开发人员只需要提交一个基本的、非常简单的 Pod YAML，Kubernetes 就可以自动给对应的 Pod 对象加上其他必要的信息，(如labels，annotations，volumes等)。而这些信息，可以是运维人员事先定义好的。这么一来，开发人员编写 Pod YAML 的门槛，就被大大降低了。PodPreset（Pod 预设置）的功能已经出现在了 v1.11 版本的 Kubernetes 中。

举个例子，现在开发人员编写了如下一个 pod.yaml 文件：

```YAML
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:
    app: website
    role: frontend
spec:
  containers:
    - name: website
      image: nginx
      ports:
        - containerPort: 80
```

这种 Pod 在生产环境里根本不能用，所以，这个时候，就可以定义一个 PodPreset 对象。在这个对象中，凡是想在上述编写的 Pod 里追加的字段，都可以预先定义好。比如这个 preset.yaml：

```YAML
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-database
spec:
  selector:
    matchLabels:
      role: frontend
  env:
    - name: DB_PORT
      value: "6379"
  volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}
```

在这个 PodPreset 的定义中：

1. 首先，是一个 **selector**。这就意味着后面这些追加的定义，**只会作用于selector所定义的、带有“role:frontend”标签的 Pod 对象**
2. 然后，定义了一组Pod的Spec里的标准字段，以及对应的值

> 比如，env里定义了DB_PORT环境变量，volumeMounts 定义了容器 Volume 的挂载目录，volumes 定义了一个 emptyDir 的 Volume。

接下来，我们假定运维人员先创建了这个 PodPreset，然后开发人员才创建 Pod：

```bash
kubectl create -f preset.yaml
kubectl create -f pod.yaml

# Pod运行起来之后，查看整个Pod的API对象
kubectl get pod website -o yaml
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:
    app: website
    role: frontend
  annotations:
    podpreset.admission.kubernetes.io/podpreset-allow-database: "resource version"
spec:
  containers:
    - name: website
      image: nginx
      volumeMounts:
        - mountPath: /cache
          name: cache-volume
      ports:
        - containerPort: 80
      env:
        - name: DB_PORT
          value: "6379"
  volumes:
    - name: cache-volume
      emptyDir: {}
```

这个时候，就可以看到：

1. 这个 Pod 里多了新添加 labels、env、volumes 和volumeMount 的定义，它们的配置跟 PodPreset 的内容一样
2. 此外，这个 Pod 还被自动加上了一个 annotation 表示这个 Pod 对象被 PodPreset 改动过

**需要说明的是，PodPreset 里定义的内容，只会在 Pod API 对象被创建之前追加在这个对象本身上，而不会影响任何 Pod 的控制器的定义。**

> 比如，我们现在提交的是一个nginx-deployment，那么这个 **Deployment 对象本身是永远不会被PodPreset改变的**，被修改的只是这个Deployment创建出来的所有 Pod。**这一点请务必区分清楚。**

如果定义了同时作用于一个 Pod 对象的多个 PodPreset，Kubernetes会**合并**（Merge）这两个 PodPreset要做的修改。而如果它们要做的修改有冲突的话，这些**冲突字段就不会被修改**。

> Kubernetes“一切皆对象”的设计思想：应用是 Pod 对象，应用的配置是 ConfigMap 对象，应用要访问的密码则是 Secret 对象，PodPreset专门用来对Pod进行批量化、自动化修改的工具对象。
