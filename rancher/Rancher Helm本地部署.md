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

#初始化完成后，可以看到tiller
# kubectl get pod -n kube-system|grep tiller
tiller-deploy-695c6f87b7-zv7cz             1/1     Running   5          47h

#查询本地仓库信息
# helm repo list
NAME            URL
stable          https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
local           http://127.0.0.1:8879/charts
rancher-stable  https://releases.rancher.com/server-charts/stable
```

# 定制生成本地helm仓库
rancher有官方的helm部署仓库，现在我们需要在本地搭建一个helm仓库

## 启动本地仓库服务
```
# mkdir -p /info/helm/rancher
# nohup helm serve --address 0.0.0.0:8879 --repo-path /info/helm/rancher &
```
添加本地仓库,本地ip为192.168.56.101
```
# helm repo add local-repo http://192.168.56.101:8879

# helm repo list
NAME            URL
stable          https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
local           http://127.0.0.1:8879/charts
rancher-stable  https://releases.rancher.com/server-charts/stable
local-repo      http://192.168.56.101:8879
```
```

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
## 生成helm包
```
# helm package rancher --save=false
Successfully packaged chart and saved it to: /info/helm/rancher/rancher-2.2.2-dirty.0aa5d3ca0.tgz
# ll
total 12
-rw-r--r-- 1 root root  797 Aug  7 14:30 index.yaml
drwxr-xr-x 3 root root   60 Aug  7 10:37 rancher
-rw-r--r-- 1 root root 4103 Aug  7 14:28 rancher-2.2.2-dirty.0aa5d3ca0.tgz
```
备注：--save=false的作用是不将tgz文件再拷贝一份到默认的local chart repo文件夹（/root/.helm/repository/local/）下，否则默认会将tgz拷贝一份到那，并检查那个目录下的index.html是否存在，不存在会报错。

## 更新索引
```
# helm repo index --url=http://192.168.56.101:8879 .

# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "local-repo" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "rancher-stable" chart repository
Update Complete. ⎈ Happy Helming!⎈

# cat index.yaml
apiVersion: v1
entries:
  rancher:
  - apiVersion: v1
    appVersion: 0aa5d3ca0-dirty
    created: 2019-08-07T14:32:10.918013067+08:00
    description: Install Rancher Server to manage Kubernetes clusters across providers.
    digest: 2a7568d3bcbc2580860487f007736236c4a9a06154a3dddc70d7dd4a3489e299
    home: https://rancher.com
    icon: https://github.com/rancher/ui/blob/master/public/assets/images/logos/welcome-cow.svg
    keywords:
    - rancher
    maintainers:
    - email: charts@rancher.com
      name: Rancher Labs
    name: rancher
    sources:
    - https://github.com/rancher/rancher
    - https://github.com/rancher/server-chart
    urls:
    - http://192.168.56.101:8879/rancher-2.2.2-dirty.0aa5d3ca0.tgz
    version: 2.2.2-dirty.0aa5d3ca0
generated: 2019-08-07T14:32:10.91714476+08:00
```
## 校验rancher本地包.
```
]# helm search rancher
NAME                    CHART VERSION           APP VERSION     DESCRIPTION
local-repo/rancher      2.2.2-dirty.0aa5d3ca0   0aa5d3ca0-dirty Install Rancher Server to manage Kubernetes clusters acro...
rancher-stable/rancher  2.2.7                   v2.2.7          Install Rancher Server to manage Kubernetes clusters acro...
```
## 部署rancher
- 首先把rancher/rancher:dev rancher/rancher-agent:dev 两个image导入到主机上。
- 执行如下指令部署
```
helm --devel --kubeconfig=/root/.kube/config install local-repo/rancher \
  --name rancher \
  --namespace cattle-system \
  --set hostname=myrancher.yingzi.com \
  --set ingress.tls.source=secret \
  --set privateCA=true 
```
备注：一定要启用--devel参数，否则会报错。

## 打开rancher
rancher部署完成后，如果要从外部访问，可以考虑如下方式：
- ingress方式：rancher默认部署了ingress服务，如果k8s已经部署ingress controller的话，可以直接基于ingress访问
- NodePort方式：删除cattle-system下的rancher服务，然后重新创建rancher服务，内容如下：
```
# cat rancher.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: rancher
    chart: rancher-2.2.2-dirty.0aa5d3ca0
    heritage: Tiller
    release: rancher
  name: rancher
  namespace: cattle-system
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: http
    nodePort: 30080
    port: 80
    protocol: TCP
    targetPort: 80
  - name: https
    nodePort: 30443
    port: 443
    protocol: TCP
    targetPort: 443
  selector:
    app: rancher
  sessionAffinity: None
  type: NodePort

```
这样就可以通过http://ip:30800访问了。
