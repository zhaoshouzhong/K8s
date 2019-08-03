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
Makefile :下载dapper，执行dapper构建操作。dapper对应的文件为Dockerfile.dapper。image构建成功后，默认的命令是ci操作。
```
cat Makefile
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
