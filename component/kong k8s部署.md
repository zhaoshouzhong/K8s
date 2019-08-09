# 前言
kong 一个api gateway，如何在k8s中部署呢？kong提供了helm，controller ingress的部署方式。本文档主要讲解如何纯手工部署kong。
# 部署kong
## 创建ns
```
# cat 00-namespace.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: kong

```
## 创建pv,给pv用
```
# cat 01-pv.yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: kong-pv
  namespace: kong
  labels:
    type: kong
spec:
  storageClassName: kong-sc
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/kong"
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: kong-pvc
  namespace: kong
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: kong-sc
---

```
备注：创建pv前，需要在主机上创建/data/kong目录
## 部署pg
```
# cat 02-postgres.yaml
apiVersion: v1
kind: Service
metadata:
  name: kong-database
  namespace: kong
spec:
  ports:
  - name: pgql
    port: 5432
    targetPort: 5432
    protocol: TCP
  selector:
    app: postgres

---

apiVersion: apps/v1  #  for k8s versions before 1.9.0 use apps/v1beta2  and before 1.8.0 use extensions/v1beta1
kind: StatefulSet
metadata:
  name: kong-database
  namespace: kong
spec:
  serviceName: "kong-database"
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: kong-database
        image: postgres:9.5
        volumeMounts:
        - name: datadir
          mountPath: /var/lib/postgresql/data
          subPath: pgdata
        env:
        - name: POSTGRES_USER
          value: kong
        - name: POSTGRES_PASSWORD
          value: kong
        - name: POSTGRES_DB
          value: kong
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        ports:
        - containerPort: 5432
      volumes:
        - name: datadir
          persistentVolumeClaim:
            claimName: kong-pvc
      # No pre-stop hook is required, a SIGTERM plus some time is all that's
      # needed for graceful shutdown of a node.
      terminationGracePeriodSeconds: 60
---
```

## 迁移数据
```
# cat 03-migration-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    io.kompose.service: kong-migration
  name: kong-migration
  namespace: kong
spec:
  containers:
  - args:
    - kong
    - migrations
    - bootstrap
    env:
    - name: KONG_PG_HOST
      value: kong-database
    - name: KONG_PG_USER
      value: kong
    - name: KONG_PG_PASSWORD
      value: kong
    - name: KONG_PG_DATABASE
      value: kong
    image: kong:1.1
    name: kong-migration
    resources: {}
  restartPolicy: OnFailure
status: {}

```
## 部署kong
```
# cat 04-deployment-kong.yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kompose.cmd: kompose convert
    kompose.version: 1.16.0 (0c01309)
  creationTimestamp: null
  labels:
    io.kompose.service: kong
  name: kong
  namespace: kong
spec:
  type: NodePort
  ports:
  - name: "8001"
    port: 8001
  - name: "8000"
    port: 8000
    targetPort: 8000
    nodePort: 31080
  - name: "8443"
    port: 8443
    targetPort: 8443
    nodePort: 31443
  - name: "5555"
    port: 5555
    targetPort: 5555
    nodePort: 31555
  selector:
    io.kompose.service: kong
status:
  loadBalancer: {}
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    kompose.cmd: kompose convert
    kompose.version: 1.16.0 (0c01309)
  creationTimestamp: null
  labels:
    io.kompose.service: kong
  name: kong
  namespace: kong
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        io.kompose.service: kong
    spec:
      containers:
      - env:
        - name: KONG_ADMIN_LISTEN
          value: 0.0.0.0:8001
        - name: KONG_PG_HOST
          value: kong-database
        - name: KONG_PROXY_LISTEN
          value: 0.0.0.0:8000
        - name: KONG_PROXY_LISTEN_SSL
          value: 0.0.0.0:8443
        - name: KONG_STREAM_LISTEN
          value: 0.0.0.0:5555
        - name: KONG_PG_USER
          value: kong
        - name: KONG_PG_PASSWORD
          value: kong
        - name: KONG_PG_DATABASE
          value: kong
        image: kong:1.1
        name: kong
        ports:
        - containerPort: 8001
        - containerPort: 8000
        - containerPort: 8443
        - containerPort: 5555
        resources: {}
      restartPolicy: Always
```
## 部署dashboard
```
# cat 05-konga-dashbaord.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: konga
  namespace: kong
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: konga
        app: konga
    spec:
      containers:
      - name: konga
        image: konga
        ports:
        - containerPort: 1337
---
apiVersion: v1
kind: Service
metadata:
  name: konga-svc
  namespace: kong
spec:
  type: NodePort
  ports:
  - name: kong-proxy
    port: 1337
    targetPort: 1337
    nodePort: 30337
    protocol: TCP
  selector:
    app: konga

```
## 校验部署
```
# kubectl get pod -n kong
NAME                     READY   STATUS      RESTARTS   AGE
kong-6c684c67c8-gd9cf    1/1     Running     0          10h
kong-database-0          1/1     Running     0          10h
kong-migration           0/1     Completed   0          10h
konga-76cd77bcb8-znwhv   1/1     Running     0          10h
# kubectl get svc -n kong
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                       AGE
kong            NodePort    10.254.17.243    <none>        8001:31448/TCP,8000:31080/TCP,8443:31443/TCP,5555:31555/TCP   10h
kong-database   ClusterIP   10.254.179.85    <none>        5432/TCP                                                      10h
konga-svc       NodePort    10.254.247.183   <none>        1337:30337/TCP                                                10h

```
## 登录portal
http://ip:30337 就可以访问了

# 定义kong
## 问题
kong 启动后，默认开启了：8000（http请求）,8443（https请求）,8001（管理端口），5555（stream端口）。默认情况下这些端口足够使用了。
如果特殊情况下需要开启新的端口，该如何处理呢？
## 定制方法
```
kubectl exec -it kong-6c684c67c8-gd9cf -n kong /bin/sh
cd /usr/local/kong
```
编写一个nginx的conf文件,为mytest.conf，内容如下
```
charset UTF-8;

server {
    server_name mygood.yingzi.com;
    listen 0.0.0.0:9090;

    access_log logs/access.log;
    error_log logs/error.log notice;

    client_body_buffer_size 8k;

    real_ip_header     X-Real-IP;
    real_ip_recursive  off;

    location / {
       proxy_pass http://mynginx.default.svc.cluster.local;
    }

}
```
备注mynginx.default.svc.cluster.local是我本地创建的一个nginx服务
```
export KONG_NGINX_HTTP_INCLUDE="/usr/local/kong/mytest.conf"
kong reload 
```
然后ntstat -ntlp 就可以看到9090端口起来了.我们测试一下
```
/ # curl -i -X GET --url http://localhost:9090/  --header 'Host: mytest.yingzi.com'
HTTP/1.1 200 OK
Server: openresty/1.13.6.2
Date: Thu, 08 Aug 2019 13:18:29 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 612
Connection: keep-alive
Last-Modified: Tue, 23 Dec 2014 16:25:09 GMT
ETag: "54999765-264"
Accept-Ranges: bytes
Access-Control-Allow-Origin: *

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```
ok，可以了。
## 改进
前面讲的仅仅是在docker容器内部的更改方法，容器重启后就丢失了。需要做到持久化。可以考虑
- 把mytest.conf放到configmap中，挂载给容器
- 把KONG_NGINX_HTTP_INCLUDE作为环境变量放到容器中
# kong其他参考资料

https://docs.konghq.com/1.2.x/getting-started/introduction/
