# 前言
rancher以helm方式部署后，默认是没有开启audit log能力的。这对系统安全是有一定影响。这篇文章讲述如何在运行的rancher集群中，重新开启audit log功能。
# Audit log
参考rancher官方资料：

https://rancher.com/docs/rancher/v2.x/en/installation/options/api-audit-log/

# 开启审计日志
参考 ###部分，这些是在原有基础上增加的内容
```
# kubectl edit deployment rancher  -n cattle-system
      containers:
      - args:
        - --http-listen-port=80
        - --https-listen-port=443
        - --add-local=auto
        env:
        - name: CATTLE_NAMESPACE
          value: cattle-system
        - name: CATTLE_PEER_SERVICE
          value: rancher
        - name: AUDIT_LEVEL  ###
          value: "1"  ###
        - name: AUDIT_LOG_MAXAGE  ###
          value: "10"  ###
        - name: AUDIT_LOG_MAXBACKUP  ###
          value: "10"  ###
        - name: AUDIT_LOG_MAXSIZE  ###
          value: "500"  ###
        image: rancher/rancher:v2.2.2
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /healthz
            port: 80
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 30
          successThreshold: 1
          timeoutSeconds: 1
        name: rancher
        ports:
        - containerPort: 80
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /healthz
            port: 80
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 30
          successThreshold: 1
          timeoutSeconds: 1
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/rancher/ssl/cacerts.pem
          name: tls-ca-volume
          readOnly: true
          subPath: cacerts.pem
        - mountPath: /var/log/auditlog  ###
          name: audit-log  ###
      - args:  ###
        - -F  ###
        - /var/log/auditlog/rancher-api-audit.log  ###
        command:  ###
        - tail  ###
        image: busybox  ###
        imagePullPolicy: Always  ###
        name: rancher-audit-log  ###
        resources: {}  ###
        terminationMessagePath: /dev/termination-log  ###
        terminationMessagePolicy: File  ###
        volumeMounts:  ###
        - mountPath: /var/log/auditlog  ###
          name: audit-log  ###
      dnsPolicy: ClusterFirst
      nodeSelector:
        node: controller
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: rancher
      serviceAccountName: rancher
      terminationGracePeriodSeconds: 30
      volumes:
      - name: tls-ca-volume
        secret:
          defaultMode: 256
          secretName: tls-ca
      - hostPath:
          path: /data/rancher/auditlog
          type: DirectoryOrCreate
        name: audit-log
     
```
然后，保持后，系统会自动新建rancher了。

