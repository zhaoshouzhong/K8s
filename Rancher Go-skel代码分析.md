# 前言
基于k8s的go-client定制crd，controller还是有些麻烦的，为了解决这个问题，rancher团队开发了go-skel代码框架，可以相对比较容易的基于k8s定制crd，controller。
go-skel官方简介：Skeleton for Rancher Go Microservices。很简单的一句话，掌握rancher的代码架构，必须要先理解go-skel。

https://github.com/rancher/go-skel

# go-skel包目录说明
```
git clone https://github.com/rancher/go-skel.git
cd go-skel
# ll
total 44
-rw-r--r-- 1 root root  1059 Jul 29 02:41 Dockerfile.dapper
-rw------- 1 root root   425 Jul 29 03:41 go.mod
-rw------- 1 root root   567 Jul 29 03:41 go.sum
-rw-r--r-- 1 root root 10174 Jul 29 02:41 LICENSE
-rw-r--r-- 1 root root  1760 Jul 29 03:12 main.go
-rw-r--r-- 1 root root   313 Jul 29 02:41 Makefile
drwxr-xr-x 2 root root    24 Jul 29 02:57 package
drwxr-xr-x 5 root root    44 Jul 29 02:41 pkg
-rw-r--r-- 1 root root   952 Jul 29 02:41 README.md
-rw-r--r-- 1 root root   766 Jul 29 02:41 README.md.in
drwxr-xr-x 2 root root   175 Jul 29 03:53 scripts
-rwxr-xr-x 1 root root  1481 Jul 29 02:44 skel.sh
```
skel.sh是它的代码生成脚本：
./skel.sh github.com/rancher/test123
That will create a folder ./test123 that will be the skeleton for a new go project expected to be at github.com/rancher/test123

go.mod 是它的依赖模块定义:%PKG%表示这部分需要被环境变量替换的部分。从定义中可以看出，它依赖于kubernetes-1.14.1的go-client代码模块。
```
module %PKG%

go 1.12

replace (
        github.com/matryer/moq => github.com/rancher/moq v0.0.0-20190404221404-ee5226d43009
)

require (
        k8s.io/api kubernetes-1.14.1
        k8s.io/apiextensions-apiserver kubernetes-1.14.1
        k8s.io/apimachinery kubernetes-1.14.1
        k8s.io/client-go kubernetes-1.14.1
        k8s.io/code-generator kubernetes-1.14.1
)
```
Makefile：定义了项目的编译打包定义。具体含义参考如下：
```
TARGETS := $(shell ls scripts) #主要动作在scripts目录下定义，包括build，ci，package等

.dapper: #dapper是一个rancher开发的docker编译打包工具，具体参考https://github.com/rancher/dapper
        @echo Downloading dapper
        @curl -sL https://releases.rancher.com/dapper/latest/dapper-`uname -s`-`uname -m` > .dapper.tmp
        @@chmod +x .dapper.tmp
        @./.dapper.tmp -v
        @mv .dapper.tmp .dapper

$(TARGETS): .dapper
        ./.dapper $@

.DEFAULT_GOAL := default

.PHONY: $(TARGETS)
```
Dockerfile.dapper:dapper的docker image构建定义。
```
FROM golang:1.12.1-alpine3.9

ARG DAPPER_HOST_ARCH # 定义构建入参，需要指定主机类型，比如amd64
ENV ARCH $DAPPER_HOST_ARCH #作为环境变量

##安装依赖的构建环境包，比如golang等
RUN apk -U add bash git gcc musl-dev docker vim less file curl wget ca-certificates
RUN go get -d golang.org/x/lint/golint && \
    git -C /go/src/golang.org/x/lint/golint checkout -b current 06c8688daad7faa9da5a0c2f163a3d14aac986ca && \
    go install golang.org/x/lint/golint && \
    rm -rf /go/src /go/pkg
RUN mkdir -p /go/src/golang.org/x && \
    cd /go/src/golang.org/x && git clone https://github.com/golang/tools && \
    git -C /go/src/golang.org/x/tools checkout -b current aa82965741a9fecd12b026fbb3d3c6ed3231b8f8 && \
    go install golang.org/x/tools/cmd/goimports
RUN rm -rf /go/src /go/pkg
RUN if [ "${ARCH}" == "amd64" ]; then \
        curl -sL https://install.goreleaser.com/github.com/golangci/golangci-lint.sh | sh -s v1.15.0; \
    fi

ENV DAPPER_ENV REPO TAG DRONE_TAG #定义环境变量
ENV DAPPER_SOURCE /go/src/%PKG%/ #设置build的代码源目录
ENV DAPPER_OUTPUT ./bin ./dist  #设置构建的可执行文件目录
ENV DAPPER_DOCKER_SOCKET true  #绑定docker.sock 到容器
ENV HOME ${DAPPER_SOURCE} #设置HOME环境变为代码源目录
WORKDIR ${DAPPER_SOURCE} #设置Workdir为代码源目录

ENTRYPOINT ["./scripts/entry"]
CMD ["ci"] #执行scripts目录下的ci脚本

```
main.go: 主入口程序，不过这个是样例代码，具体参考
```
[root@calico go-skel]# cat main.go
//go:generate go run pkg/codegen/cleanup/main.go  ##go:generate 用来标识代码生成，执行go generate指令时处理。先执行清理动作
//go:generate /bin/rm -rf pkg/generated  ##删除pkg/generated
//go:generate go run pkg/codegen/main.go  ## 运行pkg/codegen/main.go 进行代码生成

package main

import (
        "context"
        "flag"
        "fmt"
        "%PKG%/pkg/foo"  ##%PKG% 会被替换
        "%PKG%/pkg/generated/controllers/some.api.group"
        "github.com/rancher/wrangler/pkg/resolvehome"
        "github.com/rancher/wrangler/pkg/signals"
        "github.com/rancher/wrangler/pkg/start"
        "github.com/sirupsen/logrus"
        "github.com/urfave/cli"
        "k8s.io/client-go/tools/clientcmd"
        "os"
)
```
scripts目录：时间的编译和构建脚本
```
# ll scripts/
total 44
-rw-r--r-- 1 root root 566 Jul 30 16:07 boilerplate.go.txt
-rwxr-xr-x 1 root root 562 Jul 30 16:07 build
-rwxr-xr-x 1 root root  88 Jul 30 16:07 ci
-rwxr-xr-x 1 root root  63 Jul 30 16:07 default
-rwxr-xr-x 1 root root 144 Jul 30 16:07 entry
-rwxr-xr-x 1 root root 339 Jul 30 16:07 package
-rwxr-xr-x 1 root root  35 Jul 30 16:07 release
-rwxr-xr-x 1 root root  92 Jul 30 16:07 test
-rwxr-xr-x 1 root root 386 Jul 30 16:07 validate
-rwxr-xr-x 1 root root 191 Jul 30 16:07 validate-ci
-rwxr-xr-x 1 root root 490 Jul 30 16:07 version

```
pkg/apis/some.api.group/v1/types.go：定义crd类型.  some.api.group 为api组名称
```
package v1

import (
        metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +genclient ###client端代码生成
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

###定义Foo crd对象
type Foo struct {
        metav1.TypeMeta   `json:",inline"`
        metav1.ObjectMeta `json:"metadata,omitempty"`
        Spec              FooSpec `json:"spec"`
}

type FooSpec struct {
        Option bool `json:"option"` ##定义了一个Option属性
}
```
