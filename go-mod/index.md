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

我们发现新加了一行`require github.com/gin-gonic/gin v1.6.3` ，其中v1.6.3 表示gin的版本，go get 拉取依赖的原则是：

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

module名用于`import` ,可以是名字形式也可以是路径形式（例：github.com/golang/crypto）,这样可避免命名冲突。

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

### 4.6.创建全新的module推送至github并使用

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

继续在firstgomod目录下执行命令提交`firstgomod` 到github

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

上面提到，在没有明确指定依赖版本的情况下，go mod会拉取最新release tag, 若没有tag, 则拉取最新的commit生成一个自定义的版本号，当前firgomod就是这种情况（v0.0.0-20201226041745-5ea883bf90c4），版本号分三段："默认版本号-最新Commit提交时间-最新CommitID). 我们要尽量避免使用这样的依赖，因为它实际上并没有进行版本控制，依赖是不稳定的。

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

### 4.7.为firstgomod添加版本控制

打新的Release Tag

```shell
$ cd /path/to/firstgomod
$ git tag v1.0.0
$ git push --tags
Total 0 (delta 0), reused 0 (delta 0)
To github.com:Longyy/firstgomod.git
 * [new tag]         v1.0.0 -> v1.0.0
```

更新依赖

```shell
$ go get -u github.com/Longyy/firstgomod
go: github.com/Longyy/firstgomod upgrade => v1.0.0
go: downloading github.com/Longyy/firstgomod v1.0.0
```

查看go.mod

```mod
module gomod

go 1.14

require (
	github.com/Longyy/firstgomod v1.0.0
	github.com/gin-gonic/gin v1.6.3
)
```

看到firstgomod已经更新到最新的Tag v1.0.0

> 当我们自己在维护module的时候要注意使用语义化版本控制规范，在打补丁(patch)或升级的时候要从具体Tag上切分支，而不是main(或master), 这样才利于保证版本的向后兼容性。

### 4.8.给Module添加打patch

接着我们来给Module打补丁: 给函数添加注释

```shell
$ git checkout v1.0.0
$ git checkout -b feature-v1
```

在util.go中添加注释

```go
package firstgomod

//计算整数较大值
func Max(x, y int) bool {
	return x > y
}
//计算整数较小值
func Min(x, y int) bool {
	return x < y
}
```

提交patch到github

```shell
$ git commit -am 'add func comment'
$ git push -u origin feature-v1
$ git tag v1.0.1
$ git push --tags
```

更新gomod依赖

```shell
$ go get -u github.com/Longyy/firstgomod
go: github.com/Longyy/firstgomod upgrade => v1.0.1
go: downloading github.com/Longyy/firstgomod v1.0.1
```

看到依赖已经更新。

### 4.9.给Module更新次要版本

更新次要版本不应该影响module的向后兼容性，我们给firstgomod添加新函数Avg：

```go
package firstgomod

//计算整数较大值
func Max(x, y int) bool {
	return x > y
}
//计算整数较小值
func Min(x, y int) bool {
	return x < y
}
//计算平均值
func Avg(x, y float32) float32 {
	return (x + y) / 2
}
```

打tag并提交到github

```shell
$ git commit -am 'add Avg func'
$ git push
$ git tag v1.1.1
$ git push --tags
```

使用最新版

```shell
$ go get -u github.com/Longyy/firstgomod
go: github.com/Longyy/firstgomod upgrade => v1.1.1
go: downloading github.com/Longyy/firstgomod v1.1.1
```

### 4.10.给Module更新主版本

主版本更新可以不保证向后兼容性(同一项目的不同主版本可以认为是各自相互独立的项目)，这里我们给三个函数重命名来达到目的

```
package firstgomod/v2

//计算整数较大值
func IsMax(x, y int) bool {
	return x > y
}
//计算整数较小值
func IsMin(x, y int) bool {
	return x < y
}
//计算平均值
func GetAvg(x, y float32) float32 {
	return (x + y) / 2
}
```

同时为表示区分，我们将firstgomod的的module名字改为`github.com/Longyy/firstgomod/v2` 

```mod
module github.com/Longyy/firstgomod/v2

go 1.14
```

打主版本Tag 

```shell
$ git commit -am 'add v2'
$ git push
$ git tag v2.0.0
$ git push --tags
```

在gomod项目中使用新的主版本v2.0.0

```shell
$ go get -u github.com/Longyy/firstgomod
```

这里发现依赖并没有更新，原因就是`go get -u` 不会去更新依赖的主版本号，必须要手动指定

```shell
$ go get github.com/Longyy/firstgomod@v2.0.0
go get github.com/Longyy/firstgomod@v2.0.0: github.com/Longyy/firstgomod@v2.0.0: invalid version: module contains a go.mod file, so major version must be compatible: should be v0 or v1, not v2
```

这时依赖还是不能更新，错误信息告诉我们在现有go.mod依赖下，主版本号必须是v0 或 v1。

我们需要在项目中import这个新版本的依赖，并且在代码中使用

```mod
import (
	`fmt`
	`github.com/Longyy/firstgomod/v2`
	`github.com/gin-gonic/gin`
)

func main() {
	// 使用firstgomod
	fmt.Println("avg value is:", firstgomod.GetAvg(1 ,2))

	r := gin.Default()
	r.GET("/test", func(context *gin.Context) {
		context.JSON(200, gin.H{
			"msg": "ok",
		})
	})
	_ = r.Run(":8080")
}
```

然后运行`go mod tidy` ,可以看见新的版本被拉下来了

```mod

go 1.14

require (
	github.com/Longyy/firstgomod/v2 v2.0.0
	github.com/gin-gonic/gin v1.6.3
	github.com/go-playground/validator/v10 v10.4.1 // indirect
	github.com/golang/protobuf v1.4.3 // indirect
	github.com/json-iterator/go v1.1.10 // indirect
	github.com/leodido/go-urn v1.2.1 // indirect
	github.com/modern-go/concurrent v0.0.0-20180306012644-bacd9c7ef1dd // indirect
	github.com/modern-go/reflect2 v1.0.1 // indirect
	github.com/ugorji/go v1.2.2 // indirect
	golang.org/x/crypto v0.0.0-20201221181555-eec23a3978ad // indirect
	golang.org/x/sys v0.0.0-20201231184435-2d18734c6014 // indirect
	google.golang.org/protobuf v1.25.0 // indirect
	gopkg.in/yaml.v2 v2.4.0 // indirect
```

当然，一个项目的新老版本是可以同时使用的，只要在import时使用别名就好了

```go
package main

import (
	`fmt`
	`github.com/Longyy/firstgomod`
	firstgomodv2 `github.com/Longyy/firstgomod/v2`
	`github.com/gin-gonic/gin`
)

func main() {
	// 使用firstgomod
	fmt.Println("avg value is:", firstgomodv2.GetAvg(1 ,2))
	fmt.Println("Avg value is:", firstgomod.Avg(1 ,2))
  // ...
}
```

## 5.go mod如何保证项目依赖的稳定性

go.mod文件记录项目所有依赖项目及其版本信息，可有没有想过，我们依赖的这些库如果被篡改了怎么办？或者依赖库的作者删除了我们正在依赖的某些版本怎么办？

实际上, go mod使用了分布式的依赖检验方式，没有统一的依赖镜像中心，每一个项目都维护一份本地的依赖信息，是不是有点“区块链”的味道~

### 5.1.go.sum

这里go mod引入了go.sum文件协同go.mod文件进行依赖管理。

go.sum的作用类似于“借条”，里边详细记录了每一笔“借款”帐目：向谁借的，借的什么，落款签名。其结构如下：

```txt
<module> <version>[/go.mod] <hash>
```

每行记录由module名、版本、哈希值组成。这里分两种情况：

* 如果所依赖的版本使用了go mod, 则module名就是go.mod中的`module`字段
* 如果没有使用go mod, 则module名就是项目路径

比如：

```
//使用了go mod
github.com/go-playground/validator/v10 v10.4.1 h1:pH2c5ADXtd66mxoE0Zm9SUhxE20r7aM3F26W0hOn+GE=
//未使用go mod
github.com/modern-go/concurrent v0.0.0-20180306012644-bacd9c7ef1dd h1:TRLaZ9cD/w8PVh93nsPXa1VrQ6jlwL5oN8l14QlcNfg=
```

同时，还会为每个依赖版本库生成一行go.mod文件的检验值

```
github.com/gin-gonic/gin v1.6.3/go.mod h1:75u5sXoLsGZoRN5Sgbi1eraJ4GU3++wFwWzhwvtwp4M=
```

这个值主要是用在生成依赖树的时候，不必将所有的依赖库代码都拉下来，只用go.mod就行了。

如何生成依赖树呢，使用`go mod why xxx(module name)` 命令即可：

```shell
$ go mod why github.com/go-playground/validator/v10
# github.com/go-playground/validator/v10
gomod
github.com/gin-gonic/gin
github.com/gin-gonic/gin/binding
github.com/go-playground/validator/v10
```

Hash值前面都会跟一个"h1:"的前缀，表示使用的是hash算法的版本，目前只使用了一种hash算法：SHA-256.

go.mod只记录直接依赖与间接依赖（当依赖没有使用go mod时），go.sum会记录下每一笔版本信息.

当我们执行go get xxx命令时, go mod会将依赖下载到`$GOPATH/pkg/mod/cache/download` 指示的路径下，并生成v.x.y.x.zip的压缩包，同时生成该压缩包的hash值放入v.x.y.x.ziphash文件中，如果执行go get时目录中有go.mod文件，则在生成好hash值后，会同步更新go.mod和go.sum文件，将版本信息及hash值写入。

同时，为保证hash值真实可靠，go mod会向环境变量GOSUMDB指示的服务发起请求，检验该hash是否真实可靠。

### 5.2.GOSUMDB

我们可以通过设置环境变量GOSUMDB指示一个校验数据库服务，提供查询依赖包版本`哈希值` 的服务，此时go get在拉取每一个依赖时，都会去查询该依赖的检验和是否合法，如果不合法go get 将中止执行。

GOSUMDB=off即关闭此服务，go get将跳开校验，相信所有依赖库。

Google官方的`sum.golang.org`记录了所有的可公开获得的`依赖包版本`。除了使用官方的数据库，还可以指定自行搭建的数据库。

## 6.总结

本文用较长的篇幅详细说明了go mod的前世今生，以及在使用中应该注意处理的细节、常用操作的罗列，希望大家在通读后能对go的依赖包管理机制有所了解，并在工作做到心中有数、灵活运用。

欢迎留言交流，谢谢。
