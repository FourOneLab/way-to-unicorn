# Dockerfile

制作镜像的时候，`Docker` 进程通过读取 `Dockerfile` （类似 `Makefile`） 文件中指定的命令来构建镜像的每一层。

一个镜像不是说能 `run` 就完事了，需要保证如下两点：

1. 镜像体积足够小：包括 `rootfs`、安装的依赖、编译后的文件等
2. 容器权限足够小：只赋予容器内进程所必须的权限，缩小容器攻击面，减小对宿主机的影响

对于函数计算这样的场景下，以上两点就是对镜像和容器的基本要求了。

> 写 `Dockerfile` 也是在写代码，可读性、可扩展性、可维护性、简洁性、可复用性、灵活性一个都不能少。

## 镜像体积足够小

一个容器镜像是很多个**只读层**组成，每一个只读层对应Dockerfile中的一条指令（[Docker指令](https://docs.docker.com/engine/reference/builder/)而非Linux指令），每一层都是在上一层基础上的一个 `detla` 累加。

### 构建上下文

执行 `docker build` 命令时所指定的路径（通常是`.`，表示当前路径，也可以是相对路径或绝对路径）就是构建上下文，无关 `Dockerfile` 文件所在位置。可通过 `-f` 从指定文件或 `-f-` 从标准输入读取 `Dockerfile`。当前构建上下文中的文件和文件夹都会被递归的传递给 `docker daemon`。

```shell
docker build [OPTIONS] -
```

上面的连字符（`-`） 表示从标准输入中读取 `Dockerfile` 文件的内容，并且无需指定构建上下文，因为该构建上下文中只包含一个 `Dockerfile` 文件。**不推荐**，更好的做法是使用 [`.dockerignore`](https://docs.docker.com/engine/reference/builder/#dockerignore-file) 文件来指定构建是需要忽略的文件。

### 基础镜像

所有的镜像都是从一个父镜像（也称为基础镜像）开始构建的，所以基础选的好，构建镜像就很小。**最小的基础镜像 `FROM scratch`，表示为空里面啥都没有，大部分情况下，我们的应用都会对系统库有所依赖，所以在编译应用的时候需要使用静态编译将业务逻辑和依赖库一起打包在同一个二进制文件中，否则这样的容器是运行不起来的。**

> 镜像只是将代码和依赖的文件打包在一起，真正运行的时候，还是要靠宿主机的内核去执行代码，这就所以不同的操作系统、不同的CPU架构就有不同的指令集，**所以编译的时候需要特别注意这一点**。操作系统主要影响的是系统依赖库和其他依赖的安装方式，CPU架构主要影响的是指令集。

大部分情况下，我们的容器都是运行在 Linux 平台上的，当然也会有 Windows 或这 [Wasmer](https://wasmer.io/) 这样的平台，一般会选择某个 Linux 发行版作为基础镜像。

各种发行版基础镜像大小一览：（以 amd64 架构的 latest 标签版本为例，大部分云环境也都是这个架构）

> 私以为，不同发行版之间最大的区别有三个，一个是 C 语言的运行时库，另一个自带的常用软件工具，再一个是软件包管理工具。

|名称|描述|大小|
|---|---|---|
|busybox|将 Unix 中的常用工具的精简版集成到一个可执行文件中，比 GNU 工具集少，对于小型和嵌入式设备来说，已经是一个想当完整的环境|746.79KB|
|Alpine|基于 [musl libc](https://www.musl-libc.org/) 和 [busybox](https://www.busybox.net/) 组成，相比于 [busybox](https://www.busybox.net/) 的优势是有一个[软件包仓库](https://pkgs.alpinelinux.org/packages)|2.68MB|
|Debian|Linux 操作系统，有基于 GNU 协议的开源软件组成|48.1MB|
|NeuroDebian|基于 Debian 但是提供很多神经科学研究所需的软件工具|58.4MB|
|Ubuntu|基于 Debian 的 Linux 操作系统|29.9MB|
|Centos|基于 RHEL 的社区驱动的 Linux 发行版，现在已经木有了|71.7MB|
|fedora|Linux 操作系统|58.97MB|
|oraclelinux|Linux 操作系统，与 Oracle 生态强绑定|78.06MB|
|opensuse/leap|Linux 操作系统|40.44MB|
|archlinux|轻量、灵活的 Linux 操作系统|128.36MB|
|gcc|GNU 编译器集合，支持各种编程语言的编译器系统，是GNU工具链的关键组成部分|409.41MB|

各个镜像或多或少都有一个叫 `slim` 的版本，这个版本会比上面表格中的镜像大小更小一些。

不同 Linux 发行版依赖库：

|名称|描述|常用|
|---|---|---|
|[uClibc](https://uclibc.org/)|基于 [Buildroot](https://buildroot.org/) 静态编译而成|嵌入式设备|
|[glibc](https://www.gnu.org/software/libc/)|GNU发布的 libc 库是 linux 系统中最底层的 api，几乎其它任何运行库都会依赖于它|Debian|
|[musl libc](https://www.musl-libc.org/)|基于 Alpine 的 C 运行时库静态编译而成，比 glibc 更轻量级|Alpine|

### distroless

上面说的基础镜像是将整个操作系统环境进行精简，还有另一种思路就是 [`distroless`](https://github.com/GoogleContainerTools/distroless),这样的镜像中只包含应用和运行时依赖，`gcr.io/distroless/base`镜像大小为19.2MB。

> 在 `distroless` 的镜像中没有软件包管理工具，没有shell、也没有任何在标准 Linux 发行版中可用的软件工具。

这样的好处是，容器的攻击面最小，这样的坏处就是像进入容器这样的操作也是做不到的。

### 构建镜像

1. 利用镜像缓存：将不变的依赖库放在前面，经常变的代码放在后面；注意，**缓存只在宿主机上，DockerInDocker的化就没有效果了**
2. 清理不需要的文件：apt、apk、yum安装的时候会有缓存文件或相关文件

|效果|Debian类|Alpine|Centos类|
|---|---|---|---|
|不安装建议性（非必须）的依赖|`apt-get install -y -no-install-recommends`|`apk add --update --no-cache`||
|清理安装后的缓存文件|`rm -rf /var/lib/apt/lists/*`|`rm -rf /var/cache/apk/*`|`yum clean all`|

> 划重点，**安装和清理缓存需要在同一层中进行，也就是写在同一个RUN指令中**。

因为在另一层进行操作的话，其实只是覆盖了上一层的文件，真实的文件还是在的。所以在单独一层中进行任何移动、更改（包括属性、权限等）、删除文件，都会出现文件复制一个副本，从而镜像非常大的情况。

### 多阶段构建

通常分为编译阶段和运行阶段，这样可以最小化最后使用的镜像：

- 在编译阶段，将业务代码打包为一个二进制可执行文件
- 在运行阶段，使用最合适的基础镜像运行上面的二进制可执行文件

几个小的注意点：

1. 从上一个阶段拷贝内容：`COPY --from=0 ...`（阶段没有别名时）或者 `FROM golang:1.13 AS builder`，`COPY --from=builder ...`
2. 从一个已知镜像中拷贝内容：`COPY --from=nginx:latest ...`
3. 停留在某一个阶段：`docker build --target builder -t my-image:latest`，关键是这个 `--target`
4. 将前一个阶段作为新的阶段：`FROM golang:1.13 AS builder`，`FROM builder AS build 1`,`FROM builder AS build 2`

### 将镜像压缩为一层

> 实验性的功能，体验下就行了。

docker 提供 `--squash` 参数，在构建的过程中将镜像的中间层都删除，然后以当前状态保存为一个单独的镜像层。好处是显而易见的，坏处就是镜像过度压缩，太小太专用了。

## 容器权限足够小

在容器运行起来后，在以镜像为基础的只读层上增加了一个**可读写**的容器层。这个处于运行状态的容器中的所有变更都是保存在这个容器层中，包括新建文件、修改文件、删除文件等操作。

## 其他工具

buildah / podman
