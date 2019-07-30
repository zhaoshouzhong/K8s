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
