calico已经支持对pod分配固定ip，参考：

https://docs.projectcalico.org/master/reference/cni-plugin/configuration

```
annotations:
  "cni.projectcalico.org/ipAddrs": "[\"192.168.0.1\"]"
```

有两个yaml文件可以参考
- 直接在定义pod
```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
  annotations:
    cni.projectcalico.org/ipAddrs: "[\"192.168.0.1\"]"
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']

```
- stateful 中定义（固定ip一般用在stateful pod场景中。）
```
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginxip"
  replicas: 1
  template:
    metadata:
      annotations:
        cni.projectcalico.org/ipAddrs: "[\"192.168.0.1\"]"
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx  
        image: nginx:1.16.0-alpine
        ports:
        - containerPort: 80
          name: web

```
备注：注意注解的位置，否则会不生效
