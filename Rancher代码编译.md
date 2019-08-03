# 前言
如何从头开始编译rancher源码，生成image？本文章重点讲述这个问题
# 下载rancher源码
我们以rancher v2.2.2 版本为样例。
```
cd /go/src/github.com/rancher/
git clone https://github.com/rancher/rancher.git -b v2.2.2
cd rancher/
ll
total 76
drwxr-xr-x  2 root root  4096 Aug  3 16:59 app
drwxr-xr-x  4 root root    92 Aug  3 16:59 chart
drwxr-xr-x  2 root root    53 Aug  3 16:59 cleanup
-rw-r--r--  1 root root  1778 Aug  3 16:59 code-of-conduct.md
-rw-r--r--  1 root root   186 Aug  3 16:59 CONTRIBUTING.md
-rw-r--r--  1 root root  3011 Aug  3 16:59 Dockerfile.dapper
-rw-r--r--  1 root root  3067 Aug  3 16:59 keybase.md
-rw-r--r--  1 root root 10175 Aug  3 16:59 LICENSE
-rw-r--r--  1 root root  5294 Aug  3 16:59 main.go
-rw-r--r--  1 root root   411 Aug  3 16:59 Makefile
drwxr-xr-x  3 root root   165 Aug  3 16:59 package
drwxr-xr-x 48 root root  4096 Aug  3 16:59 pkg
-rw-r--r--  1 root root  4233 Aug  3 16:59 README_1_6.md
-rw-r--r--  1 root root  4287 Aug  3 16:59 README.md
drwxr-xr-x  2 root root   164 Aug  3 16:59 rke-templates
drwxr-xr-x  5 root root  4096 Aug  3 16:59 scripts
drwxr-xr-x  6 root root   109 Aug  3 16:59 server
drwxr-xr-x  3 root root    91 Aug  3 16:59 tests
drwxr-xr-x 10 root root   157 Aug  3 16:59 vendor
-rw-r--r--  1 root root  4686 Aug  3 16:59 vendor.conf

```
rancher的编译打包采用makefile 和 dapper方式
## Makefile
Makefile :下载dapper，执行dapper构建操作。dapper对应的文件为Dockerfile.dapper。image构建成功后，默认的命令是ci操作。
```
# cat Makefile
TARGETS := $(shell ls scripts) 

.dapper:
        @echo Downloading dapper
        @curl -sL https://releases.rancher.com/dapper/latest/dapper-`uname -s`-`uname -m` > .dapper.tmp
        @@chmod +x .dapper.tmp
        @./.dapper.tmp -v
        @mv .dapper.tmp .dapper

$(TARGETS): .dapper
        ./.dapper $@

trash: .dapper
        ./.dapper -m bind trash

trash-keep: .dapper
        ./.dapper -m bind trash -k

deps: trash
 
.DEFAULT_GOAL := ci

.PHONY: $(TARGETS)
```
- .DEFAULT_GOAL := ci ,这说明如果没有指定make选项的话，默认执行ci这个target。但是ci这个target在哪里呢？
- TARGETS := $(shell ls scripts) ，这里指定TARGETS，scripts目录下包括 ci 的target
- $(TARGETS): .dapper ，表示ci依赖于 .dapper 这个target。本身它这个target 执行 ./.dapper $@ ，就是执行Dockerfile.dapper的内容。
- .dapper 这个target 则执行下载dapper，赋予执行权限等操作
- 因此，这个Makefile的操作就是下载dapper，执行Dockerfile.dapper的image构建
## Dockerfile.dapper
查看具体的内容可以知道，这个文件就是安装rancher构建依赖的环境包。
最后执行ENTRYPOINT ["./scripts/entry"] 和 CMD ["ci"] 。看对应的脚本，其动作就是执行 scripts/ci脚本。
为了减少构建时间，我们对这个文件进行调整。拆成两部分，一部分是image生成，另外一部分就是ci脚本执行。
### 构建Dockerfile
```
# cat file/Dockerfile
FROM ubuntu:18.04
# FROM arm=armhf/ubuntu:16.04 arm64=arm64v8/ubuntu:18.04

#ARG DAPPER_HOST_ARCH
ENV HOST_ARCH=amd64  ARCH=amd64
ENV CATTLE_HELM_VERSION v2.10.0-rancher10

# 注意设置代理，否则下载包会出问题
ENV http_proxy=http://172.xx.xx.108:1080 
ENV https_proxy=http://172.xx.xx.108:1080

RUN apt-get update && \
    apt-get install -y gcc ca-certificates git wget curl vim less file xz-utils unzip && \
    rm -f /bin/sh && ln -s /bin/bash /bin/sh
RUN curl -sLf https://github.com/rancher/machine-package/releases/download/v0.15.0-rancher5-3/docker-machine-${ARCH}.tar.gz | tar xvzf - -C /usr/bin

ENV GOLANG_ARCH_amd64=amd64 GOLANG_ARCH_arm=armv6l GOLANG_ARCH_arm64=arm64 GOLANG_ARCH=GOLANG_ARCH_${ARCH} \
    GOPATH=/go PATH=/go/bin:/usr/local/go/bin:${PATH} SHELL=/bin/bash

RUN wget -O - https://storage.googleapis.com/golang/go1.11.linux-${!GOLANG_ARCH}.tar.gz | tar -xzf - -C /usr/local && \
    go get github.com/rancher/trash && curl -L https://raw.githubusercontent.com/alecthomas/gometalinter/v3.0.0/scripts/install.sh | sh

ENV DOCKER_URL_amd64=https://get.docker.com/builds/Linux/x86_64/docker-1.10.3 \
    DOCKER_URL_arm=https://github.com/rancher/docker/releases/download/v1.10.3-ros1/docker-1.10.3_arm \
    DOCKER_URL_arm64=https://github.com/rancher/docker/releases/download/v1.10.3-ros1/docker-1.10.3_arm64 \
    DOCKER_URL=DOCKER_URL_${ARCH}

ENV HELM_URL_amd64=https://github.com/rancher/helm/releases/download/${CATTLE_HELM_VERSION}/helm \
    HELM_URL_arm64=https://github.com/rancher/helm/releases/download/${CATTLE_HELM_VERSION}/helm-arm64 \
    HELM_URL=HELM_URL_${ARCH} \
    TILLER_URL_amd64=https://github.com/rancher/helm/releases/download/${CATTLE_HELM_VERSION}/tiller \
    TILLER_URL_arm64=https://github.com/rancher/helm/releases/download/${CATTLE_HELM_VERSION}/tiller-arm64 \
    TILLER_URL=TILLER_URL_${ARCH}

RUN curl -sLf ${!HELM_URL} > /usr/bin/helm && \
    curl -sLf ${!TILLER_URL} > /usr/bin/tiller && \
    chmod +x /usr/bin/helm /usr/bin/tiller && \
    helm init -c --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts  && \
    helm plugin install https://github.com/rancher/helm-unittest

RUN wget -O - ${!DOCKER_URL} > /usr/bin/docker && chmod +x /usr/bin/docker

# FIXME: Update to Rancher RKE when released
ENV RKE_URL_amd64=https://github.com/rancher/rke/releases/download/v0.2.0-rc8/rke_linux-amd64 \
    RKE_URL_arm64=https://github.com/rancher/rke/releases/download/v0.2.0-rc8/rke_linux-arm64 \
    RKE_URL=RKE_URL_${ARCH}

RUN wget -O - ${!RKE_URL} > /usr/bin/rke && chmod +x /usr/bin/rke
ENV KUBECTL_URL=https://storage.googleapis.com/kubernetes-release/release/v1.11.0/bin/linux/${ARCH}/kubectl
RUN wget -O - ${KUBECTL_URL} > /usr/bin/kubectl && chmod +x /usr/bin/kubectl

RUN apt-get update && \
    apt-get install -y tox python3.7 python3-dev python3.7-dev libffi-dev libssl-dev
```    
然后我们生成一个image
```
docker build -t rancher:build.v2.2.2 -f file/Dockerfile .
```
### 构建Dockerfile.dapper
```
# cat Dockerfile.dapper
FROM rancher:build.v2.2.2
# FROM arm=armhf/ubuntu:16.04 arm64=arm64v8/ubuntu:18.04


ENV http_proxy=""
ENV https_proxy=""

ENV HELM_HOME /root/.helm
ENV DAPPER_ENV REPO TAG DRONE_TAG
ENV DAPPER_SOURCE /go/src/github.com/rancher/rancher/
ENV DAPPER_OUTPUT ./bin ./dist
ENV DAPPER_DOCKER_SOCKET true
ENV TRASH_CACHE ${DAPPER_SOURCE}/.trash-cache
ENV HOME ${DAPPER_SOURCE}
WORKDIR ${DAPPER_SOURCE}

ENTRYPOINT ["./scripts/entry"]
CMD ["ci"]
```
# 调整image生成Dockerfile
build 动作：包括server端代码，agent代码的构建
```
# ll scripts/build*
-rwxr-xr-x 1 root root 172 Aug  2 16:46 scripts/build
-rwxr-xr-x 1 root root 361 Aug  2 16:46 scripts/build-agent
-rwxr-xr-x 1 root root 763 Aug  2 16:46 scripts/build-server
```

分析scripts下的脚本，package脚本
```
# cat scripts/package
#!/bin/bash
set -e
source $(dirname $0)/version

ARCH=${ARCH:-"amd64"}
SUFFIX=""
[ "${ARCH}" != "amd64" ] && SUFFIX="_${ARCH}"

cd $(dirname $0)/../package ###这步是进入package目录下

TAG=${TAG:-${VERSION}${SUFFIX}}
REPO=${REPO:-rancher}

if echo $TAG | grep -q dirty; then
    TAG=dev
fi

if [ -n "$DRONE_TAG" ]; then
    TAG=$DRONE_TAG
fi

cp ../bin/rancher ../bin/agent .

IMAGE=${REPO}/rancher:${TAG}
AGENT_IMAGE=${REPO}/rancher-agent:${TAG}

if [ ${ARCH} == arm64 ]; then
    sed -i -e '$a\' -e 'ENV ETCD_UNSUPPORTED_ARCH=arm64' Dockerfile
fi

docker build --build-arg VERSION=${TAG} --build-arg ARCH=${ARCH} -t ${IMAGE} .  ###执行构建rancher-server image
docker build --build-arg VERSION=${TAG} --build-arg ARCH=${ARCH} -t ${AGENT_IMAGE} -f Dockerfile.agent . ###执行构建rancher agent image
echo ${IMAGE} > ../dist/images
echo ${AGENT_IMAGE} >> ../dist/images
echo Built ${IMAGE}
echo Built ${AGENT_IMAGE} # 默认是不显示的
echo

cd ../bin
mkdir -p /tmp/system-charts && git clone --branch master https://github.com/rancher/system-charts /tmp/system-charts
TAG=$TAG REPO=${REPO} go run ../pkg/image/export/main.go /tmp/system-charts $IMAGE $AGENT_IMAGE
```
调整dockerfile,分为server和agent的的dockerfile
```
# ll
total 36
-rw-r--r-- 1 root root 3982 Aug  2 19:40 Dockerfile
-rw-r--r-- 1 root root 1809 Aug  2 19:41 Dockerfile.agent
-rw-r--r-- 1 root root  283 Apr 17 04:17 entrypoint.sh
-rwxr-xr-x 1 root root  385 Apr 17 04:17 kubectl-shell.sh
-rwxr-xr-x 1 root root 8479 Apr 17 04:17 run.sh
-rwxr-xr-x 1 root root  420 Apr 17 04:17 share-root.sh
-rwxr-xr-x 1 root root 1366 Apr 17 04:17 shell-setup.sh
drwxr-xr-x 2 root root  148 Apr 17 04:17 windows
```
主要是增加代理，两个文件都需要处理。示例如下：
```
# cat Dockerfile
FROM ubuntu:18.04
RUN apt-get update && apt-get install -y git curl ca-certificates unzip xz-utils && \
    useradd rancher && \
    mkdir -p /var/lib/rancher/etcd /var/lib/cattle && \
    chown -R rancher /var/lib/rancher /var/lib/cattle /usr/local/bin
RUN mkdir /root/.kube && \
    ln -s /usr/bin/rancher /usr/bin/kubectl && \
    ln -s /var/lib/rancher/management-state/cred/kubeconfig.yaml /root/.kube/config && \
    ln -s /usr/bin/rancher /usr/bin/reset-password && \
    ln -s /usr/bin/rancher /usr/bin/ensure-default-admin && \
    rm -f /bin/sh && ln -s /bin/bash /bin/sh
WORKDIR /var/lib/rancher

ARG ARCH=amd64
ENV CATTLE_HELM_VERSION v2.10.0-rancher10
ENV CATTLE_MACHINE_VERSION v0.15.0-rancher6-1
ENV LOGLEVEL_VERSION v0.1.2
ENV TINI_VERSION v0.18.0
ENV TELEMETRY_VERSION v0.5.3

# 增加代理设置
ENV http_proxy=http://192.168.1.106:1080 
ENV https_proxy=http://192.168.1.106:1080 

.......... 省略
# 取消代理设置
ENV http_proxy="" 
ENV https_proxy=""


ENV SSL_CERT_DIR /etc/rancher/ssl
VOLUME /var/lib/rancher

ENTRYPOINT ["entrypoint.sh"]

```
修改ci脚本,如果想节省时间，减少test的处理，可以如下处理：
```
# cat scripts/ci
#!/bin/bash
set -x

cd $(dirname $0)

./validate
./build
#./test
./package
./chart/ci

```
# 执行构建
```
# pwd
/go/src/github.com/rancher/rancher
# make 
```
构建生成后，会生成对应的image
```
# docker images
REPOSITORY                                            TAG                 IMAGE ID            CREATED              SIZE
rancher/rancher-agent                                 dev                 17bdc51e98a7        About a minute ago   298MB
rancher/rancher                                       dev                 54f2e4c2a984        About a minute ago   472MB
rancher                                               HEAD                9f5dbc019dbe        10 minutes ago       1.55GB
rancher                                               OZoUfGm             ca6404ddb801        6 hours ago          1.48GB
rancher                                               build.v2.2.2        83eccea8b9ea        27 hours ago         1.41GB

```
# 启动rancher server
```
docker run --rm -it -p 80:80 -p 443:443 rancher/rancher:dev
```
浏览器打开就可以看到rancher的管理页面了。
