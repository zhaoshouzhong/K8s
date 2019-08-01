# 理解k8s的crd，controller定制原理，参考：

https://itnext.io/how-to-create-a-kubernetes-custom-controller-using-client-go-f36a7a7536cc

https://medium.com/@trstringer/create-kubernetes-controllers-for-core-and-custom-resources-62fc35ad64a3

# 环境初始化
os版本：centos7,内核 4.14.123
部署go：
```
wget https://dl.google.com/go/go1.12.7.linux-amd64.tar.gz
tar xvzf go1.12.7.linux-amd64.tar.gz -C /
mkdir /gosrc

vi /etc/profile #增加：
export GOROOT=/go
export GOPATH=/gosrc
export PATH=$PATH:/$GOROOT/bin

source /etc/profile
#测试一下go，出现如下显示，则表示ok
# go version
go version go1.12.7 linux/amd64
```
# 下载样例代码
```
mkdir -p /gosrc/src/github.com/trstringer
cd /gosrc/src/github.com/trstringer
git clone https://github.com/trstringer/k8s-controller-custom-resource.git
cd k8s-controller-custom-resource/
# ll
total 36
-rw-r--r-- 1 root root  4356 Jul 28 21:08 controller.go
drwxr-xr-x 2 root root    29 Jul 28 21:08 crd
drwxr-xr-x 2 root root    37 Jul 28 21:08 example
-rw-r--r-- 1 root root 11162 Jul 28 21:08 Gopkg.lock
-rw-r--r-- 1 root root  2466 Jul 28 21:08 Gopkg.toml
-rw-r--r-- 1 root root   934 Jul 28 21:08 handler.go
-rw-r--r-- 1 root root  4074 Jul 28 21:08 main.go
drwxr-xr-x 4 root root    32 Jul 28 21:08 pkg
-rw-r--r-- 1 root root  1112 Jul 28 21:08 README.md

```
下载的样例代码中，默认包含了client端生成的代码，我们把它删除，自己验证一下手工生产代码桩的过程。
```
# cd pkg/
# ll
total 0
drwxr-xr-x 3 root root 24 Jul 28 21:08 apis
drwxr-xr-x 5 root root 55 Jul 28 21:08 client
# rm -rf client/

```
# 定义generate脚本
脚本参考上面的连接，但是上面的连接脚本有些bug，我们调整为如下：
脚本名称为gen.sh，赋予执行权限：chmod 755 gen.sh
```
#!/usr/bin/bash
set +x

# ROOT_PACKAGE :: the package (relative to $GOPATH/src) that is the target for code generation
ROOT_PACKAGE="github.com/trstringer/k8s-controller-custom-resource"
# CUSTOM_RESOURCE_NAME :: the name of the custom resource that we're generating client code for
CUSTOM_RESOURCE_NAME="myresource"
# CUSTOM_RESOURCE_VERSION :: the version of the resource
CUSTOM_RESOURCE_VERSION="v1"

# retrieve the code-generator scripts and bins
go get -u k8s.io/code-generator
cd $GOPATH/src/k8s.io/code-generator

# run the code-generator entrypoint script
./generate-groups.sh all "$ROOT_PACKAGE/pkg/client" "$ROOT_PACKAGE/pkg/apis" "$CUSTOM_RESOURCE_NAME:$CUSTOM_RESOURCE_VERSION"

# view the newly generated files
tree tree --charset ASCII  $GOPATH/src/$ROOT_PACKAGE/pkg/client
# pkg/client/
# ├── clientset
# │   └── versioned
# │       ├── clientset.go
# │       ├── doc.go
# │       ├── fake
# │       │   ├── clientset_generated.go
# │       │   ├── doc.go
# │       │   └── register.go
# │       ├── scheme
# │       │   ├── doc.go
# │       │   └── register.go
# │       └── typed
# │           └── myresource
# │               └── v1
# │                   ├── doc.go
# │                   ├── fake
# │                   │   ├── doc.go
# │                   │   ├── fake_myresource_client.go
# │                   │   └── fake_myresource.go
# │                   ├── generated_expansion.go
# │                   ├── myresource_client.go
# │                   └── myresource.go
# ├── informers
# │   └── externalversions
# │       ├── factory.go
# │       ├── generic.go
# │       ├── internalinterfaces
# │       │   └── factory_interfaces.go
# │       └── myresource
# │           ├── interface.go
# │           └── v1
# │               ├── interface.go
# │               └── myresource.go
# └── listers
#     └── myresource
#         └── v1
#             ├── expansion_generated.go
#             └── myresource.go
#
# 16 directories, 22 files
```
安装tree 软件包
```
 yum install -y tree
 alias tree='tree --charset ASCII'
``` 
# 桩代码生成
go get时需要翻墙才能下载一些包，因此，需要设置本地代理(ip：端口号为我本地主机的代理，根据实际情况进行替换)：
```
export http_proxy="http://172.xx.xx.52:1080"
export https_proxy="http://172.xx.xx.52:1080"
```
运行gen.sh,出现错误。原来是缺少依赖的包
```
# ./gen.sh
package k8s.io/code-generator: build constraints exclude all Go files in /root/go/src/k8s.io/code-generator
cmd/defaulter-gen/main.go:48:2: cannot find package "github.com/spf13/pflag" in any of:
        /go/src/github.com/spf13/pflag (from $GOROOT)
        /root/go/src/github.com/spf13/pflag (from $GOPATH)
cmd/defaulter-gen/args/args.go:23:2: cannot find package "k8s.io/gengo/args" in any of:
        /go/src/k8s.io/gengo/args (from $GOROOT)
        /root/go/src/k8s.io/gengo/args (from $GOPATH)
cmd/defaulter-gen/args/args.go:24:2: cannot find package "k8s.io/gengo/examples/defaulter-gen/generators" in any of:
        /go/src/k8s.io/gengo/examples/defaulter-gen/generators (from $GOROOT)
        /root/go/src/k8s.io/gengo/examples/defaulter-gen/generators (from $GOPATH)
cmd/defaulter-gen/main.go:51:2: cannot find package "k8s.io/klog" in any of:
        /go/src/k8s.io/klog (from $GOROOT)
        /root/go/src/k8s.io/klog (from $GOPATH)
cmd/client-gen/generators/scheme/generator_for_scheme.go:28:2: cannot find package "k8s.io/gengo/generator" in any of:
        /go/src/k8s.io/gengo/generator (from $GOROOT)
        /root/go/src/k8s.io/gengo/generator (from $GOPATH)
cmd/client-gen/types/helpers.go:25:2: cannot find package "k8s.io/gengo/namer" in any of:
        /go/src/k8s.io/gengo/namer (from $GOROOT)
        /root/go/src/k8s.io/gengo/namer (from $GOPATH)
cmd/client-gen/generators/scheme/generator_for_scheme.go:30:2: cannot find package "k8s.io/gengo/types" in any of:
        /go/src/k8s.io/gengo/types (from $GOROOT)
        /root/go/src/k8s.io/gengo/types (from $GOPATH)
cmd/deepcopy-gen/args/args.go:24:2: cannot find package "k8s.io/gengo/examples/deepcopy-gen/generators" in any of:
        /go/src/k8s.io/gengo/examples/deepcopy-gen/generators (from $GOROOT)
        /root/go/src/k8s.io/gengo/examples/deepcopy-gen/generators (from $GOPATH)
./gen.sh: line 21: tree: command not found

```
安装依赖的包
```
go get github.com/spf13/pflag
go get k8s.io/gengo/args
go get k8s.io/gengo/examples/defaulter-gen/generators
go get k8s.io/klog
go get k8s.io/gengo/generator
go get k8s.io/gengo/namer
go get k8s.io/gengo/types
go get k8s.io/gengo/examples/deepcopy-gen/generators
```
继续执行gen.sh,显示错误信息
```
# ./gen.sh
package k8s.io/code-generator: build constraints exclude all Go files in /gosrc/src/k8s.io/code-generator
Generating deepcopy funcs
F0728 21:30:49.145124   18119 deepcopy.go:885] Hit an unsupported type invalid type for invalid type, from github.com/trstringer/k8s-controller-custom-resource/pkg/apis/myresource/v1.MyResource
/gosrc/src/github.com/trstringer/k8s-controller-custom-resource/pkg/client [error opening dir]

0 directories, 0 files
```
根据错误信息，说这个未支持的类型，也就是说可能是类型定义go文件存在编译问题。
```
# cd pkg/apis/myresource/v1/
# ll
total 16
-rw-r--r-- 1 root root   70 Jul 28 21:08 doc.go
-rw-r--r-- 1 root root 1108 Jul 28 21:08 register.go
-rw-r--r-- 1 root root 1196 Jul 28 21:08 types.go
-rw-r--r-- 1 root root 3002 Jul 28 21:08 zz_generated.deepcopy.go
```
我们直接build一下，看看是否可以build成功.
```
# go build
register.go:4:2: cannot find package "k8s.io/apimachinery/pkg/apis/meta/v1" in any of:
        /go/src/k8s.io/apimachinery/pkg/apis/meta/v1 (from $GOROOT)
        /gosrc/src/k8s.io/apimachinery/pkg/apis/meta/v1 (from $GOPATH)
register.go:5:2: cannot find package "k8s.io/apimachinery/pkg/runtime" in any of:
        /go/src/k8s.io/apimachinery/pkg/runtime (from $GOROOT)
        /gosrc/src/k8s.io/apimachinery/pkg/runtime (from $GOPATH)
register.go:6:2: cannot find package "k8s.io/apimachinery/pkg/runtime/schema" in any of:
        /go/src/k8s.io/apimachinery/pkg/runtime/schema (from $GOROOT)
        /gosrc/src/k8s.io/apimachinery/pkg/runtime/schema (from $GOPATH)
```
果然是build问题，存在缺失的包：
```
go get k8s.io/apimachinery/pkg/apis/meta/v1
go get k8s.io/apimachinery/pkg/runtime
go get k8s.io/apimachinery/pkg/runtime/schema

go build #再没有显示错误了
```
再次执行gen.sh
```
# ./gen.sh
package k8s.io/code-generator: build constraints exclude all Go files in /gosrc/src/k8s.io/code-generator
Generating deepcopy funcs
Generating clientset for myresource:v1 at github.com/trstringer/k8s-controller-custom-resource/pkg/client/clientset
Generating listers for myresource:v1 at github.com/trstringer/k8s-controller-custom-resource/pkg/client/listers
Generating informers for myresource:v1 at github.com/trstringer/k8s-controller-custom-resource/pkg/client/informers
/gosrc/src/github.com/trstringer/k8s-controller-custom-resource/pkg/client
|-- clientset
|   `-- versioned
|       |-- clientset.go
|       |-- doc.go
|       |-- fake
|       |   |-- clientset_generated.go
|       |   |-- doc.go
|       |   `-- register.go
|       |-- scheme
|       |   |-- doc.go
|       |   `-- register.go
|       `-- typed
|           `-- myresource
|               `-- v1
|                   |-- doc.go
|                   |-- fake
|                   |   |-- doc.go
|                   |   |-- fake_myresource_client.go
|                   |   `-- fake_myresource.go
|                   |-- generated_expansion.go
|                   |-- myresource_client.go
|                   `-- myresource.go
|-- informers
|   `-- externalversions
|       |-- factory.go
|       |-- generic.go
|       |-- internalinterfaces
|       |   `-- factory_interfaces.go
|       `-- myresource
|           |-- interface.go
|           `-- v1
|               |-- interface.go
|               `-- myresource.go
`-- listers
    `-- myresource
        `-- v1
            |-- expansion_generated.go
            `-- myresource.go

16 directories, 22 files
```
# 编译打包整个样例代码
前面我们已经把controller client的桩代码生成了，我们后面则进行整个项目的打包
```
# pwd
/gosrc/src/github.com/trstringer/k8s-controller-custom-resource
go get -d -v ./... #下载所有依赖的包。前面的过程也可以采用这种方法
# go build
# ll
total 38668
-rw-r--r-- 1 root root     4356 Jul 28 21:08 controller.go
drwxr-xr-x 2 root root       29 Jul 28 21:08 crd
drwxr-xr-x 2 root root       37 Jul 28 21:08 example
-rwxr-xr-x 1 root root     2299 Jul 28 21:40 gen.sh
-rw-r--r-- 1 root root    11162 Jul 28 21:08 Gopkg.lock
-rw-r--r-- 1 root root     2466 Jul 28 21:08 Gopkg.toml
-rw-r--r-- 1 root root      934 Jul 28 21:08 handler.go
-rwxr-xr-x 1 root root 39553204 Jul 28 21:50 k8s-controller-custom-resource
-rw-r--r-- 1 root root     4074 Jul 28 21:08 main.go
drwxr-xr-x 4 root root       32 Jul 28 21:38 pkg
-rw-r--r-- 1 root root     1112 Jul 28 21:08 README.md
###可以看到我们已经成功编译 k8s-controller-custom-resource 这个可执行文件了
```
创建crd文件
```
cd crd
[root@k8s02 crd]# ll
total 4
-rw-r--r-- 1 root root 235 Jul 28 21:08 myresource.yaml
# kubectl apply -f myresource.yaml
```
启动k8s-controller-custom-resource
```
# ./k8s-controller-custom-resource
INFO[0000] Successfully constructed k8s client
INFO[0000] Controller.Run: initiating
INFO[0000] Controller.Run: cache sync complete
INFO[0000] Controller.runWorker: starting
INFO[0000] Controller.processNextItem: start
```
生成myresource数据
```
# cd example/
[root@k8s02 example]# ll
total 4
-rw-r--r-- 1 root root 129 Jul 28 21:08 example-myresource.yaml
# kubectl apply -f example-myresource.yaml
myresource.trstringer.com/example-myresource created
```
查看controller日志,可以看到controler检测到新建信息：
```
# ./k8s-controller-custom-resource
INFO[0000] Successfully constructed k8s client
INFO[0000] Controller.Run: initiating
INFO[0000] Controller.Run: cache sync complete
INFO[0000] Controller.runWorker: starting
INFO[0000] Controller.processNextItem: start
INFO[0082] Add myresource: default/example-myresource
INFO[0082] Controller.processNextItem: object created detected: default/example-myresource
INFO[0082] TestHandler.ObjectCreated
INFO[0082] Controller.runWorker: processing next item
INFO[0082] Controller.processNextItem: start

```
删除数据
```
# kubectl delete -f example-myresource.yaml
myresource.trstringer.com "example-myresource" deleted
```
controller日志,可以看到controller检测到删除事件：
```
# ./k8s-controller-custom-resource
INFO[0000] Successfully constructed k8s client
INFO[0000] Controller.Run: initiating
INFO[0000] Controller.Run: cache sync complete
INFO[0000] Controller.runWorker: starting
INFO[0000] Controller.processNextItem: start
INFO[0082] Add myresource: default/example-myresource
INFO[0082] Controller.processNextItem: object created detected: default/example-myresource
INFO[0082] TestHandler.ObjectCreated
INFO[0082] Controller.runWorker: processing next item
INFO[0082] Controller.processNextItem: start
INFO[0176] Delete myresource: default/example-myresource
INFO[0176] Controller.processNextItem: object deleted detected: default/example-myresource
INFO[0176] TestHandler.ObjectDeleted
INFO[0176] Controller.runWorker: processing next item
INFO[0176] Controller.processNextItem: start

```
至此，基于k8s的controller样例代码编写、generate client桩代码，build，运行已经ok。
# 其他坑
- go 1.11版本后引入go module概念，下载样例代码后，如果启用了export GO111MODULE=on ，则后面如果在重新生成client桩代码，则会报错，提示代码路径找不到。
