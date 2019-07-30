# 理解k8s的crd，controller定制原理，参考：

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

GOPATH=/root/go

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
tree $GOPATH/src/$ROOT_PACKAGE/pkg/client
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
# 桩代码生成
go get时需要翻墙才能下载一些包，因此，需要设置本地代理(ip：端口号为我本地主机的代理，根据实际情况进行替换)：
```
export http_proxy="http://172.19.68.52:1080"
export https_proxy="http://172.19.68.52:1080"
```
