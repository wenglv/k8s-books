###### 监控集群节点基础指标

node_exporter  https://github.com/prometheus/node_exporter 

分析：

- 每个节点都需要监控，因此可以使用DaemonSet类型来管理node_exporter
- 添加节点的容忍配置
- 挂载宿主机中的系统文件信息

```bash
$ cat node-exporter.ds.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitor
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.0.1
        args:
        - --web.listen-address=$(HOSTIP):9100
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --collector.filesystem.ignored-mount-points=^/(dev|proc|sys|var/lib/docker/.+)($|/)
        - --collector.filesystem.ignored-fs-types=^(autofs|binfmt_misc|cgroup|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|mqueue|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|sysfs|tracefs)$
        ports:
        - containerPort: 9100
        env:
        - name: HOSTIP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        resources:
          requests:
            cpu: 150m
            memory: 180Mi
          limits:
            cpu: 150m
            memory: 180Mi
        securityContext:
          runAsNonRoot: true
          runAsUser: 65534
        volumeMounts:
        - name: proc
          mountPath: /host/proc
        - name: sys
          mountPath: /host/sys
        - name: root
          mountPath: /host/root
          mountPropagation: HostToContainer
          readOnly: true
      tolerations:
      - operator: "Exists"
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: dev
        hostPath:
          path: /dev
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
```



创建node-exporter服务

```bash
$ kubectl apply -f node-exporter.ds.yaml

$ kubectl -n monitor get po -owide
node-exporter-djcqx           1/1     Running   0          110s   172.21.51.68


$ curl 172.21.51.143:9100/metrics
```



问题来了，如何添加到Prometheus的target中？

- 配置一个Service，后端挂载node-exporter的服务，把Service的地址配置到target中
  - 带来新的问题，target中无法直观的看到各节点node-exporter的状态
- 把每个node-exporter的服务都添加到target列表中
  - 带来新的问题，集群节点的增删，都需要手动维护列表
  - target列表维护量随着集群规模增加

