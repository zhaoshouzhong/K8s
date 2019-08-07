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
## 创建service
```
# cat svc.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster1
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
    name: client
  - port: 16379
    targetPort: 16379
    name: gossip
  selector:
    statefulset.kubernetes.io/pod-name: redis-cluster1-0
    #app: redis-cluster

---
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster2
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
    name: client
  - port: 16379
    targetPort: 16379
    name: gossip
  selector:
    statefulset.kubernetes.io/pod-name: redis-cluster2-0
    #app: redis-cluster

---
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster3
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
    name: client
  - port: 16379
    targetPort: 16379
    name: gossip
  selector:
    statefulset.kubernetes.io/pod-name: redis-cluster3-0
    #app: redis-cluster

---
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster4
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
    name: client
  - port: 16379
    targetPort: 16379
    name: gossip
  selector:
    statefulset.kubernetes.io/pod-name: redis-cluster4-0
    #app: redis-cluster

---
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster5
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
    name: client
  - port: 16379
    targetPort: 16379
    name: gossip
  selector:
    statefulset.kubernetes.io/pod-name: redis-cluster5-0
    #app: redis-cluster

---
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster6
spec:
  type: ClusterIP
  ports:
  - port: 6379
    targetPort: 6379
    name: client
  - port: 16379
    targetPort: 16379
    name: gossip
  selector:
    statefulset.kubernetes.io/pod-name: redis-cluster6-0
    #app: redis-cluster
```
# 创建redis集群
```
kubectl exec -it redis-cluster1-0 -- redis-cli --cluster create --cluster-replicas 1 $(kubectl get pods -l app=redis-cluster -o jsonpath='{range.items[*]}{.status.podIP}:6379 ' )
```
# 测试redis集群
部署完成后，执行如下指令进行测试验证
```
# kubectl exec -it redis-cluster1-0 -- redis-cli cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:7
cluster_my_epoch:1
cluster_stats_messages_ping_sent:42367
cluster_stats_messages_pong_sent:42565
cluster_stats_messages_fail_sent:10
cluster_stats_messages_auth-ack_sent:1
cluster_stats_messages_sent:84943
cluster_stats_messages_ping_received:42565
cluster_stats_messages_pong_received:42358
cluster_stats_messages_fail_received:2
cluster_stats_messages_auth-req_received:1
cluster_stats_messages_received:84926

# for x in $(seq 1 6); do echo "redis-cluster$x-0"; kubectl exec redis-cluster$x-0 -- redis-cli role; echo; done
redis-cluster1-0
master
59262
10.100.236.143
6379
59262

redis-cluster2-0
slave
10.100.236.144
6379
connected
59248

redis-cluster3-0
master
59304
10.100.236.145
6379
59304

redis-cluster4-0
slave
10.100.236.140
6379
connected
59262

redis-cluster5-0
master
59248
10.100.236.141
6379
59248

redis-cluster6-0
slave
10.100.236.142
6379
connected
59304
```
则表示集群正常运行了。

# 非固定ip方案
可以参考

https://rancher.com/blog/2019/deploying-redis-cluster/

这种方案部署就简单多了。如果是简单测试用，可以考虑这种方式。
