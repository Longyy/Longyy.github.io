# Go mod原理一次讲透


## 1.前言

说起项目依赖管理工具大家一点都不陌生，像Java的Maven、Python的Pip、Php的Composer、Nodejs的Npm这些工具大家都耳熟能详。项目依赖管理工具是项目的基石之一，它的好坏往往深刻影响着项目开发运行的效率、稳定性、扩展性、安全性等重要指标。今天我们来聊一聊Golang的包管理工具Go mod.

在此之前，我们必须要熟悉两个重要的环境变量GOROOT与GOPATH.

`GOROOT` 指示GO的安装位置，其中有GO的原生类库(SDK)，在编译GO时需要用到；

`GOPATH` 指示项目代码和依赖包的位置；必须设置，但是可以随项目不同而重新设置；GOPATH下有三个目录：

- `src` 存放项目代码或依赖包源码；

- `bin` 存放由`go install` 或 `go get` 编译产生的二进制文件，为方便执行，最好将`$GOPATH/bin` 加入PATH路径； 

- `pkg` 存放预编译产生的文件用于加速编译过程，通常我们不需要关心这个目录，它由GO自动管理；

  ```shell
  $ echo $GOPATH
  /Users/longyongyu/Development/go/workspace
  $ tree ./pkg/darwin_amd64
  pkg/darwin_amd64
  └── github.com
      ├── gogo
      │   └── protobuf
      │       ├── gogoproto.a
      │       ├── jsonpb.a
      │       └── proto.a
      └── golang
          └── protobuf
              └── proto.a
   $ ls -l ./bin
   total 533296
  -rwxr-xr-x  1 longyongyu  staff   8961980 10 21 11:59 db2struct
  -rwxr-xr-x  1 longyongyu  staff  18795324  9  9 18:47 dlv
  -rwxr-xr-x  1 longyongyu  staff   5871480  6 22  2020 goimports
  -rwxr-xr-x  1 longyongyu  staff   6315432  6 22  2020 golint
  $ tree -L 2 ./src
  ├── go.uber.org
  │   ├── atomic
  │   ├── multierr
  │   └── zap
  ├── golang.org
  │   └── x
  ├── google.golang.org
  │   ├── genproto
  │   ├── grpc
  │   └── protobuf
  ```

查看当前项目的GO环境变量可以用`go env`

```shell
$ go env
GO111MODULE="on"
GOARCH="amd64"
GOBIN=""
GOCACHE="/Users/longyongyu/Library/Caches/go-build"
GOENV="/Users/longyongyu/Library/Application Support/go/env"
GOEXE=""
GOFLAGS=""
GOHOSTARCH="amd64"
GOHOSTOS="darwin"
GOINSECURE=""
GONOPROXY="xxx" //去掉了敏感信息
GONOSUMDB="yyy" //去掉了敏感信息
GOOS="darwin"
GOPATH="/Users/longyongyu/Development/go/workspace"
GOPRIVATE="zzz" //去掉了敏感信息
GOPROXY="https://goproxy.cn"
GOROOT="/usr/local/go"
GOSUMDB="sum.golang.org"
GOTMPDIR=""
GOTOOLDIR="/usr/local/go/pkg/tool/darwin_amd64"
GCCGO="gccgo"
AR="ar"
CC="clang"
CXX="clang++"
CGO_ENABLED="1"
GOMOD="/dev/null"
CGO_CFLAGS="-g -O2"
CGO_CPPFLAGS=""
CGO_CXXFLAGS="-g -O2"
CGO_FFLAGS="-g -O2"
CGO_LDFLAGS="-g -O2"
PKG_CONFIG="pkg-config"
GOGCCFLAGS="-fPIC -m64 -pthread -fno-caret-diagnostics -Qunused-arguments -fmessage-length=0\
-fdebug-prefix-map=/var/folders/dg/b4mzy9tx1ln1y7bkblz5lvsc0000gn/T/go-build462069466=/tmp/go-build\
-gno-record-gcc-switches -fno-common"
```

要更改GO环境变量可以用`go env -w` 命令，如：

```shell
$ go env -w GOPROXY=test.com
```

## 2.Go依赖管理工具发展

Go的包管理工具几经调整，从最开始的gopath、govender到后来的go mod, 以致go mod成为自v1.13以后的事实标准，都反映出了大家对于一款好用的包管理工具的急切渴望。下面列出其主要工具的发展过程：

<img src="/Users/longyongyu/Development/Blog/content/posts/Go/img/image-dev2.png" alt="image-20201222002754640" style="zoom:33%;" />

V1.5之前，使用原生的`gopath` 进行包管理，要求项目源码与依赖包必须位于$GOPATH/src目录下；最大的问题，不能进行依赖包的多版本管理；

V1.5, 引入`govendor` ，GO环境变量需设置`GO15VENDOREXPERIMENT=1` 在项目根目录下自动生成vendor目录并存放依赖包, GO在编译时会优先去vendor目录查询依赖；vendor对依赖包进行了分类：

| 状态      | 缩写 | 含义                                               |
| --------- | ---- | -------------------------------------------------- |
| +local    | l    | 本地包，即项目自身的包组织                         |
| +external | e    | 外部包，即被 $GOPATH 管理，但不在 vendor 目录下    |
| +vendor   | v    | 已被 govendor 管理，即在 vendor 目录下             |
| +std      | s    | 标准库中的包                                       |
| +unused   | u    | 未使用的包，即包在 vendor 目录下，但项目并没有用到 |
| +missing  | m    | 代码引用了依赖包，但该包并没有找到                 |
| +program  | p    | 主程序包，意味着可以编译为执行文件                 |
| +outside  |      | 外部包和缺失的包                                   |
| +all      |      | 所有的包                                           |

v1.9, 引入`godep` , 在govendor的基础上，记录了依赖包的版本信息存放于项目根目录的Godep目录下，需要与vendor目录一起提交至代码库；godep常用命令：

* `go get -u -v github.com/tools/godep` 安装godep；
* `go get github.com/globalsign/mgo` 下载并安装依赖包；
* `import github.com/globalsign/mgo` 引用依赖包；
* `godep go build main.go` 编译项目；
* `godep save` 更新Godeps/Godeps.json文件；
* `godep restore` 拉取Godeps/Godeps.json文件定义的依赖包至$GOPATH/src目录下；

[Kubernetes](https://github.com/kubernetes/kubernetes/tree/release-1.13)早期版本就是用godep管理依赖的。

v1.11, 社区推出了`go mod` 方案，目前已成为GO的主流包管理工具，且大部分GO项目已支持go mod. go mod具有几个特点：

* 项目代码摆脱了对`$GOPATH/src` 目录的依赖，在此之外的任何目录都可以使用go mod;
* Goproxy代理协议，可以使用代理拉取依赖；
* go mod的Tag必须遵循[语义化版本控制](https://semver.org/lang/zh-CN/), 如果没有则将忽略Tag, 转而根据commit时间与commit hash值生成一个临时的版本号；
* 模块缓存。同一Module版本的数据只缓存一份，所有其他模块共享，目前所有模块版本数据均缓存在$GOPATH/pkg/mod下，可以使用`go clean -modcache` 清理所有Module缓存

## 3.各个工具主要特点对比

通常，一个好用的包管理工具应该具备下面几项能力

* 依赖管理
* 依赖包版本控制
* 包管理平台
* 私有化部署
* 代码包复用
* 支持代理（众所周知的原因）

| 工具     | 依赖管理 | 依赖包版本控制 | 包管理平台 | 私有化部署 | 代码包复用 | 支持代理 |
| -------- | -------- | -------------- | ---------- | ---------- | ---------- | -------- |
| gopath   | **Y**    | N              | N          | N          | N          | N        |
| govendor | **Y**    | **Y**          | N          | N          | N          | N        |
| godep    | **Y**    | **Y**          | N          | N          | N          | N        |
| go mod   | **Y**    | **Y**          | **Y**      | **Y**      | **Y**      | **Y**    |



## 4.使用go mod

### 4.1.设置环境变量

要使用go mod，go版本必须在v1.11及以上；然后设置必要的环境变量（小知识：使用命令`go help environment` 可以查看go支持的所有环境变量及其用法）

```shell
$ go env -w GO111MODULE=on
$ go env -w GOPROXY=https://goproxy.cn,direct //使用七牛云的代理服务
$ go env -w GOPATH=/Users/longyongyu/Development/go/workspace
```

`go env -w` 会把配置写到`go env GOENV` 所配置的文件中，在我的macOS系统是`/Users/longyongyu/Library/Application Support/go/env` 文件。

```shell
$ cat /Users/longyongyu/Library/Application\ Support/go/env
GO111MODULE=on
GOPATH=/Users/longyongyu/Development/go/workspace
GOPROXY=https://goproxy.cn,direct
```

设置`GO111MODULE=on` 表示启用go mod， 此时`go get` ` go build` `go run` 等命令将被go mod接管，并在运行这些命令的时候自动维护`go.mod` `go.sum` 文件。

`goproxy` 用于设置依赖的代理服务。

### 4.2.初始化项目

```shell
$ mkdir gomod
$ cd gomod
$ go mod init gomod
go: creating new go.mod: module gomod
$ cat go.mod
module gomod

go 1.14
```

### 4.3.增加依赖

我们使用`go get` 命令为新项目添加依赖`gin`:

```shell
$ go get github.com/gin-gonic/gin
go: github.com/gin-gonic/gin upgrade => v1.6.3
$ cat go.mod
module gomod

go 1.14

require github.com/gin-gonic/gin v1.6.3

```

我们发现新加了一行require github.com/gin-gonic/gin v1.6.3` ，其中v1.6.3 表示gin依赖的版本，go get 拉取依赖的原则是：

* 若指定了版本，则拉取指定版本代码；
* 若未指定版本，则拉取最新的release tag.
* 若无tag, 则拉取最新的commit.

### 4.4.升级依赖

go get 升级依赖的方法：

* `go get -u xxx` 将升级到最新次要版本，或者修订版本（x.y.z中，x为主版本，y为次要版本，z为修订版本）
* `go get -u=patch xxx` 将升级到最新的修订版本
* `go get xxx@version` 将升级到指定的version

`go list -m -u all`列出可以升级的package

```shell
$ go list -m -u all
gomod
github.com/davecgh/go-spew v1.1.1
github.com/gin-contrib/sse v0.1.0
github.com/gin-gonic/gin v1.6.3
github.com/go-playground/assert/v2 v2.0.1
github.com/go-playground/locales v0.13.0
github.com/go-playground/universal-translator v0.17.0
github.com/go-playground/validator/v10 v10.2.0 [v10.4.1]
github.com/golang/protobuf v1.3.3 [v1.4.3]
github.com/google/gofuzz v1.0.0 [v1.2.0]
github.com/json-iterator/go v1.1.9 [v1.1.10]
github.com/leodido/go-urn v1.2.0 [v1.2.1]
github.com/mattn/go-isatty v0.0.12
github.com/modern-go/concurrent v0.0.0-20180228061459-e0a39a4cb421 [v0.0.0-20180306012644-bacd9c7ef1dd]
github.com/modern-go/reflect2 v0.0.0-20180701023420-4b7aa43c6742 [v1.0.1]
github.com/pmezard/go-difflib v1.0.0
...
```

`go get -u xxx` 升级指定package

`go get -u ` 升级所有package

### 4.5.go.mod文件的四个关键字

* `module` 指定包的名字
* `require` 指定具体依赖项
* `replace` 替换依赖项
* `exclude` 忽略依赖项

module名用于`import` 可以是名字形式也可以是路径形式（例：github.com/golang/crypto）,这样可避免命名冲突。

replace 用于替换无法直接拉取的依赖（国内朋友深有体会），该语句可以将依赖包用github上的库（或者本地私有库）进行替换。如

```
replace (
    golang.org/x/crypto v0.0.0-20190313024323-a1f597ede03a => github.com/golang/crypto v0.0.0-20190313024323-a1f597ede03a
)
```

exclude用于忽略某些依赖的版本，比如有的依赖的版本有问题。

```
exclude github.com/SermoDigital/jose v0.9.1
```

查看一个完整的示例

```
module github.com/example/project

require (
    github.com/SermoDigital/jose v0.0.0-20180104203859-803625baeddc
    github.com/google/uuid v1.1.0
)

exclude github.com/SermoDigital/jose v0.9.1

replace github.com/google/uuid v1.1.0 => git.coolaj86.com/coolaj86/uuid.go v1.1.1
```

### 4.6.提交module到github并使用

这里我们演示如何创建和提交一个新的module, 然后做为本地项目的依赖项使用。

```shell
$ mkdir firstgomod
$ cd firstgomod
$ go mod init github.com/Longyy/firstgomod
go: creating new go.mod: module github.com/Longyy/firstgomod
$ touch util.go
```

我们在`util.go` 中实现两个方法，用来比较两个整数的大小

```go
package firstgomod

func Max(x, y int) bool {
	return x > y
}
func Min(x, y int) bool {
	return x < y
}
```

看一下go.mod的内容

```
module github.com/Longyy/firstgomod

go 1.14
```

继续在firstgomod目录下执行命令提交firstgomod`到github

```shell
$ git init
$ touch .gitignore
$ echo '.idea/' >> .gitignore # 我IDE用的goland，.idea信息应该加入.gitignore
$ touch README.md
$ echo '# firstgomod' >> README.md
$ git add .
$ git commit -m 'my first go module'
$ git remote add origin git@github.com:Longyy/firstgomod.git
$ git branch -M main # github已推荐将主分支设置为main(而非master)
# 先在远端先创建firstgomod项目...，然后将本地代码推送至远端
$ git push origin main
```

一个全新的module制作完成，下面我们回到gomod项目来使用它

```shell
$ cd gomod
$ go get github.com/Longyy/firstgomod
go: downloading github.com/Longyy/firstgomod v0.0.0-20201226041745-5ea883bf90c4
go: github.com/Longyy/firstgomod upgrade => v0.0.0-20201226041745-5ea883bf90c4
```

在gomod/main.go文件中使用

```go
package main

import (
	`fmt`

	`github.com/Longyy/firstgomod`
	`github.com/gin-gonic/gin`
)

func main() {
	// 使用firstgomod
	fmt.Println("max value is:", firstgomod.Max(1 ,2))

	r := gin.Default()
	r.GET("/test", func(context *gin.Context) {
		context.JSON(200, gin.H{
			"msg": "ok",
		})
	})
	_ = r.Run(":8080")
}
```





新建`main.go` 编写下面的代码

```go
package main

import (
	`github.com/gin-gonic/gin`
)

func main() {
	//todo
	r := gin.Default()
	r.GET("/test", func(context *gin.Context) {
		context.JSON(200, gin.H{
			"msg": "ok",
		})
	})
	_ = r.Run(":8080")
}
```



## 5.Go mod 与Go get使用




