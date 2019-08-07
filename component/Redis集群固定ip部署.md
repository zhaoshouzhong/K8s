# 前言
Redis在k8s部署后，存在一个问题：如果物理机重新启动，则redis的POD ip会发送改变，导致集群信息连接失败。
目前网上大部分资料讲如何部署在k8s集群中redis，但是没有考虑这种情况。
# 解决思路
利用calico可以给POD设定固定ip能力来解决这个问题。
# 部署方案
## 创建volume
创建6个pv，每个pv绑定一个POD。persistentVolumeReclaimPolicy: Delete。策略可以根据自己需要选择Retain。
```
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: redis-pv1
  labels:
    type: redis
spec:
  storageClassName: redis-test
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/redis-test01"
  persistentVolumeReclaimPolicy: Delete

---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: redis-pv2
  labels:
    type: redis
spec:
  storageClassName: redis-test
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/redis-test02"
  persistentVolumeReclaimPolicy: Delete

---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: redis-pv3
  labels:
    type: redis
spec:
  storageClassName: redis-test
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/redis-test03"
  persistentVolumeReclaimPolicy: Delete
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: redis-pv4
  labels:
    type: redis
spec:
  storageClassName: redis-test
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/redis-test04"
  persistentVolumeReclaimPolicy: Delete

---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: redis-pv5
  namespace: test2
  labels:
    type: redis
spec:
  storageClassName: redis-test
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/redis-test05"
  persistentVolumeReclaimPolicy: Delete
---
kind: PersistentVolume
apiVersion: v1
metadata:
  name: redis-pv6
  namespace: test2
  labels:
    type: redis
spec:
  storageClassName: redis-test
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/redis-test06"
  persistentVolumeReclaimPolicy: Delete
```
## 创建configmap
```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster
data:
  update-node.sh: |
    #!/bin/sh
    REDIS_NODES="/data/nodes.conf"
    sed -i -e "/myself/ s/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/${POD_IP}/" ${REDIS_NODES}
    exec "$@"
  redis.conf: |+
    cluster-enabled yes
    cluster-require-full-coverage no
    cluster-node-timeout 15000
    cluster-config-file /data/nodes.conf
    cluster-migration-barrier 1
    appendonly yes
    protected-mode no

```
## 部署Redis集群
需要部署6个实例，每个实例指定一个固定IP。
```
# cat pod1.yaml

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster1
spec:
  serviceName: redis-cluster1
  replicas: 1
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
      annotations:
        cni.projectcalico.org/ipAddrs: "[\"10.100.236.140\"]"
    spec:
      containers:
      - name: redis
        image: redis:5.0.1-alpine
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
      volumes:
      - name: conf
        configMap:
          name: redis-cluster
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
      storageClassName: redis-test
```
```
# cat pod2.yaml

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster2
spec:
  serviceName: redis-cluster2
  replicas: 1
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
      annotations:
        cni.projectcalico.org/ipAddrs: "[\"10.100.236.141\"]"
    spec:
      containers:
      - name: redis
        image: redis:5.0.1-alpine
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
      volumes:
      - name: conf
        configMap:
          name: redis-cluster
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
      storageClassName: redis-test
```
```
# cat pod3.yaml

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster3
spec:
  serviceName: redis-cluster3
  replicas: 1
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
      annotations:
        cni.projectcalico.org/ipAddrs: "[\"10.100.236.142\"]"
    spec:
      containers:
      - name: redis
        image: redis:5.0.1-alpine
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
      volumes:
      - name: conf
        configMap:
          name: redis-cluster
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
      storageClassName: redis-test
```
```
# cat pod4.yaml

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster4
spec:
  serviceName: redis-cluster4
  replicas: 1
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
      annotations:
        cni.projectcalico.org/ipAddrs: "[\"10.100.236.143\"]"
    spec:
      containers:
      - name: redis
        image: redis:5.0.1-alpine
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
      volumes:
      - name: conf
        configMap:
          name: redis-cluster
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
      storageClassName: redis-test
```
```
# cat pod5.yaml

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster5
spec:
  serviceName: redis-cluster5
  replicas: 1
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
      annotations:
        cni.projectcalico.org/ipAddrs: "[\"10.100.236.144\"]"
    spec:
      containers:
      - name: redis
        image: redis:5.0.1-alpine
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
      volumes:
      - name: conf
        configMap:
          name: redis-cluster
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
      storageClassName: redis-test
```
```
# cat pod6.yaml

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster6
spec:
  serviceName: redis-cluster6
  replicas: 1
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
      annotations:
        cni.projectcalico.org/ipAddrs: "[\"10.100.236.145\"]"
    spec:
      containers:
      - name: redis
        image: redis:5.0.1-alpine
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
      volumes:
      - name: conf
        configMap:
          name: redis-cluster
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
      storageClassName: redis-test
```
