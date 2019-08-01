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
scripts目录：编译和构建脚本
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
pkg/foo/controller.go: controller类，定义对crd的处理
```
# cat pkg/foo/controller.go: controller
package foo

import (
        "context"
        v1 "%PKG%/pkg/apis/some.api.group/v1"  ##%PKG%为需要替换的变量
        foocontroller "%PKG%/pkg/generated/controllers/some.api.group/v1"
)

type Controller struct {
        foos foocontroller.FooController
}

func Register(
        ctx context.Context,
        foos foocontroller.FooController) {

        controller := &Controller{
                foos: foos,
        }

        foos.OnChange(ctx, "foo-handler", controller.OnFooChange)
        foos.OnRemove(ctx, "foo-handler", controller.OnFooRemove)
}
##OnFooChange响应对象的创建，修改操作。不过这个方法有问题。后面会讲到
func (c *Controller) OnFooChange(key string, foo *v1.Foo) (*v1.Foo, error) {
        //change logic, return original foo if no changes

        fooCopy := foo.DeepCopy()
        //make changes to fooCopy
        return c.foos.Update(fooCopy)
}
##OnFooRemove 响应对象的删除操作。这个方法也是有问题的，后面会讲到
func (c *Controller) OnFooRemove(key string, foo *v1.Foo) (*v1.Foo, error) {
        //remove logic, return original foo if no changes

        fooCopy := foo.DeepCopy()
        //make changes to fooCopy
        return c.foos.Update(fooCopy)
}

```
# 定制CRD属性和api组
代码默认的crd属性只有一个属性：Option，默认的api组为some.api.group。这显然不能满足我们需要，需要需要变更如下：
- Foo属性为：Message   string `json:"message"` ; SomeValue *int32 `json:"someValue"` 两个属性
- api组为 mytest.api.group

需要改动的代码点如下：
- pkg/apis/yingzi.api.group/v1/types.go：
```
type FooSpec struct {
        //Option bool `json:"option"`
        Message   string `json:"message"`
        SomeValue *int32 `json:"someValue"`
}
```
- main.go:
```
import (
        "context"
        "flag"
        "fmt"
        "%PKG%/pkg/foo"
        "%PKG%/pkg/generated/controllers/some.api.group" ==> "%PKG%/pkg/generated/controllers/mytest.api.group"
        "github.com/rancher/wrangler/pkg/resolvehome"
        "github.com/rancher/wrangler/pkg/signals"
        "github.com/rancher/wrangler/pkg/start"
        "github.com/sirupsen/logrus"
        "github.com/urfave/cli"
        "k8s.io/client-go/tools/clientcmd"
        "os"
)
```
```
func run(c *cli.Context) {
        flag.Parse()

        logrus.Info("Starting controller")
        ctx := signals.SetupSignalHandler(context.Background())

        kubeconfig, err := resolvehome.Resolve(c.String("kubeconfig"))
        if err != nil {
                logrus.Fatalf("Error resolving home dir: %s", err.Error())
        }
        masterurl := c.String("masterurl")

        cfg, err := clientcmd.BuildConfigFromFlags(masterurl, kubeconfig)
        if err != nil {
                logrus.Fatalf("Error building kubeconfig: %s", err.Error())
        }

        foos, err := some.NewFactoryFromConfig(cfg)===>foos, err := mytest.NewFactoryFromConfig(cfg)
        if err != nil {
                logrus.Fatalf("Error building sample controllers: %s", err.Error())
        }

        foo.Register(ctx, foos.Some().V1().Foo())===>foo.Register(ctx, foos.Mytest().V1().Foo())

        if err := start.All(ctx, 2, foos); err != nil {
                logrus.Fatalf("Error starting: %s", err.Error())
        }

        <-ctx.Done()
}
```
- skel.sh
```
FILES="
./Dockerfile.dapper
./.dockerignore
./.golangci.json
./.drone.yml
./.gitignore
./LICENSE
./main.go
./Makefile
./package/Dockerfile
./README.md.in
./scripts/boilerplate.go.txt
./scripts/build
./scripts/ci
./scripts/entry
./scripts/package
./scripts/release
./scripts/test
./scripts/validate
./scripts/validate-ci
./scripts/version
./pkg/apis/some.api.group/v1/types.go ===>./pkg/apis/mytest.api.group/v1/types.go
./pkg/codegen/cleanup/main.go
./pkg/codegen/main.go
./pkg/foo/controller.go
./pkg/foo/controller_test.go
./go.mod
"
```
- pkg/apis/some.api.group目录改为 pkg/apis/mytest.api.group
- pkg/codegen/main.go
```
package main

import (
        "os"
        "%PKG%/pkg/apis/some.api.group/v1" ===>"%PKG%/pkg/apis/mytest.api.group/v1"
        "github.com/rancher/wrangler/pkg/controller-gen"
        "github.com/rancher/wrangler/pkg/controller-gen/args"
)

func main() {
        os.Unsetenv("GOPATH")
        controllergen.Run(args.Options{
                OutputPackage: "%PKG%/pkg/generated",
                Boilerplate:   "scripts/boilerplate.go.txt",
                Groups: map[string]args.Group{
                        "some.api.group": { ===> "mytest.api.group":
                                Types: []interface{}{
                                        v1.Foo{},
                                },
                                GenerateTypes: true,
                        },
                },
        })
}
```
- pkg/foo/controller.go
```

import (
        "context"
        v1 "%PKG%/pkg/apis/some.api.group/v1" ===>v1 "%PKG%/pkg/apis/mytest.api.group/v1"
        foocontroller "%PKG%/pkg/generated/controllers/some.api.group/v1" ===>foocontroller "%PKG%/pkg/generated/controllers/mytest.api.group/v1"
)
```
```
func (c *Controller) OnFooChange(key string, foo *v1.Foo) (*v1.Foo, error) {
        //change logic, return original foo if no changes

        fooCopy := foo.DeepCopy()
        //make changes to fooCopy
        return c.foos.Update(fooCopy)
}

func (c *Controller) OnFooRemove(key string, foo *v1.Foo) (*v1.Foo, error) {
        //remove logic, return original foo if no changes

        fooCopy := foo.DeepCopy()
        //make changes to fooCopy
        return c.foos.Update(fooCopy)
}
```
===> 调整为：
```
func (c *Controller) OnFooChange(key string, foo *v1.Foo) (*v1.Foo, error) {
	//change logic, return original foo if no changes

	fooCopy := foo.DeepCopy()
	if (fooCopy == nil ) {

		return nil,nil
	}
	//make changes to fooCopy
	log.Println("OnFooChange:" + key)
	return c.foos.Update(fooCopy)

}

func (c *Controller) OnFooRemove(key string, foo *v1.Foo) (*v1.Foo, error) {
	//remove logic, return original foo if no changes

	fooCopy := foo.DeepCopy()
	if (fooCopy == nil ) {

		return nil,nil
	}
	//make changes to fooCopy
	log.Println("OnFooRemove:" + key)
	return c.foos.Update(fooCopy)

}
```
- pkg/foo/controller_test.go
```
import (
        "%PKG%/pkg/apis/some.api.group/v1" ===> %PKG%/pkg/apis/mytest.api.group/v1"
        fooFakes "%PKG%/pkg/generated/controllers/some.api.group/v1/fakes"===>fooFakes "%PKG%/pkg/generated/controllers/mytest.api.group/v1/fakes"
        "github.com/stretchr/testify/assert"
        "testing"
)

```
- scripts/build
```
#!/bin/bash
set -x

source $(dirname $0)/version

cd $(dirname $0)/..

mkdir -p bin
if [ "$(uname)" = "Linux" ]; then
    OTHER_LINKFLAGS="-extldflags -static -s"
fi
###新增如下代理配置，这样go get可以翻墙拉包了
export http_proxy="http://172.xx.xx.52:1080"
export https_proxy="http://172.xx.xx.52:1080"
```
- scripts/validate
```
#!/bin/bash
set -e

cd $(dirname $0)/..

echo Running validation

PACKAGES="$(go list ./...)"

if ! command -v golangci-lint; then
    echo Skipping validation: no golangci-lint available
    exit
fi
set -x
echo Running validation

###新增如下代理配置，这样go get可以翻墙拉包了
export http_proxy="http://172.xx.xx.52:1080"
export https_proxy="http://172.xx.xx.52:1080"

```
- scripts/validate-ci
```
#!/bin/bash
set -x

cd $(dirname $0)/..

###新增如下代理配置，这样go get可以翻墙拉包了
export http_proxy="http://172.xx.xx.52:1080"
export https_proxy="http://172.xx.xx.52:1080"

```
- 为了方便调测，减少build时间，可以考虑调整如下：
1：所有脚本打开 set -x 

2：scripts/ci 脚本可以考虑去掉 ./test  ./validate  ./validate-ci  这三条指令。

# 生成样例代码：
```
export GOPROXY=https://goproxy.io
export http_proxy="http://172.xx.xx.52:1080"
export https_proxy="http://172.xx.xx.52:1080"

./skel.sh github.com/rancher/test123
```
成功后，在生成一个test123目录，可执行程序唯一该目录下的bin目录下。
# 定义CRD文件
定义crd文件：
```
# cat crd.yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: foos.mytest.api.group
spec:
  group: mytest.api.group
  version: v1
  names:
    kind: Foo
    plural: foos
  scope: Namespaced
```
```
kubectl apply -f crd.yaml
```
启动test123程序，可以看到类似如下信息：
```
time="2019-07-31T15:57:20+08:00" level=info msg="Starting controller"
time="2019-07-31T15:57:20+08:00" level=info msg="Starting yingzi.api.group/v1, Kind=Foo controller"
```
备注：必须先创建crd文件，否则会报错

创建crd实例：
```
cat example.yaml
apiVersion: mytest.api.group/v1
kind: Foo
metadata:
  name: example-foo
spec:
  message: hello world
  someValue: 13

```
```
kubectl apply -f example.yaml
```
可以看到controller日志：
```
2019/07/31 15:57:33 OnFooChange:default/example-foo
2019/07/31 15:57:33 OnFooChange:default/example-foo
```
删除实例：
```
kubectl delete -f example.yaml
```
对应controller日志：
```
2019/07/31 15:57:40 OnFooChange:default/example-foo
2019/07/31 15:57:40 OnFooRemove:default/example-foo
```
