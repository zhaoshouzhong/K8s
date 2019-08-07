# 前言
rancher 本地编译打包后，会生成对应的image，和helm包
```
]# docker images
REPOSITORY                                                TAG                 IMAGE ID            CREATED             SIZE
rancher/rancher-agent                                     dev                 17bdc51e98a7        3 days ago          298MB
rancher/rancher                                           dev                 54f2e4c2a984        3 days ago          472MB

# pwd
/go/src/github.com/rancher/rancher/bin/chart/dev
# ll
total 8
-rw-r--r-- 1 root root 4143 Aug  3 18:36 rancher-2.2.2-dirty.0aa5d3ca0.tgz
```
解压开rancher-2.2.2-dirty.0aa5d3ca0.tgz，可以看到chart的定义文件
```
# ll rancher
total 8
-rwxr-xr-x 1 root root  459 Aug  3 18:36 Chart.yaml
drwxr-xr-x 2 root root  222 Aug  5 11:44 templates
-rwxr-xr-x 1 root root 2999 Aug  3 18:36 values.yaml
```
# 修改chart默认配置
## values.yaml
```
auditLog:
  destination: sidecar
  hostPath: /var/log/rancher/audit/
  level: 0
  maxAge: 1
  maxBackup: 1
  maxSize: 100

```
rancher默认是不开启审计日志的，可以考虑打开,调整为==》
```
auditLog:
  destination: sidecar
  hostPath: /var/log/rancher/audit/
  level: 1
  maxAge: 10
  maxBackup: 10
  maxSize: 500

```
```
# Override rancher image location for Air Gap installs
rancherImage: rancher/rancher
# rancher/rancher image tag. https://hub.docker.com/r/rancher/rancher/tags/
# Defaults to .Chart.appVersion
# rancherImageTag: v2.0.7
````
这部分定义的是rancher image，需要改为我们前面build的image,调整为==》
```
# Override rancher image location for Air Gap installs
rancherImage: rancher/rancher
# rancher/rancher image tag. https://hub.docker.com/r/rancher/rancher/tags/
# Defaults to .Chart.appVersion
rancherImageTag: dev
````
其他的参数根据需要调整

# 部署helm
```
https://get.helm.sh/helm-v2.14.1-linux-amd64.tar.gz
tar xvzf helm-v2.14.1-linux-amd64.tar.gz
cd linux-amd64
mv helm /usr/local/bin/

#创建sa
kubectl -n   kube-system create serviceaccount tiller

kubectl create  clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller

#安装socat，否则会报错
yum install -y socat

#初始化helm，指向阿里云的image和仓库，否则需要翻墙
helm init --service-account tiller \
--tiller-image registry.cn-shanghai.aliyuncs.com/rancher/tiller:v2.14.1 \
--stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts \
```

# 定制生成本地helm仓库
```
mkdir -p /info/helm/rancher
# 把前面解压后的rancher目录复制到该目录下
# tree
.
`-- rancher
    |-- Chart.yaml
    |-- templates
    |   |-- clusterRoleBinding.yaml
    |   |-- deployment.yaml
    |   |-- _helpers.tpl
    |   |-- ingress.yaml
    |   |-- issuer-letsEncrypt.yaml
    |   |-- issuer-rancher.yaml
    |   |-- NOTES.txt
    |   |-- serviceAccount.yaml
    |   `-- service.yaml
    `-- values.yaml

```

