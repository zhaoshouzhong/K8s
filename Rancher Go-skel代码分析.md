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

main.go:
