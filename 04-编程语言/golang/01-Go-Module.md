# Go Module

go.mod 文件生成之后，会被 go toolchain 掌控维护，在执行：

- go run
- go build
- go get：将会使用 Git 等代码工具远程获取代码包，并自动完成编译和安装到 `GOPATH/bin` 和 `GOPATH/pkg` 目录下
- go mod
  - init：初始化
  - download：在手动修改go.mod文件后，手动更新项目的依赖关系
  - tidy：与download类似，但是会移除掉go.mod文件中没有被使用的require模块

等各类命令时自动修改和维护 go.mod 文件中的依赖内容。

可以通过 Go Modules 引入远程依赖包，如 Git Hub 中开源的 Go 开发工具包。

但可能会由于网络环境问题，在拉取 GitHub 中的开发依赖包时，有时会失败，可以使用七牛云搭建的 GOPROXY，方便在开发中更好地拉取远程依赖包。

在项目目录下执行以下命令即可配置新的 GOPROXY：`go env -w GOPROXY=https://goproxy.cn,direct`。

go.mod文件中的关键字：

- require：为项目引入版本是 vX.Y.Z 的依赖包，该依赖包可以在开发中引入使用
- replace：替换依赖模块
- exclude：忽略依赖模块
