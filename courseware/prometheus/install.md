##### Prometheus安装

基于go开发， https://github.com/prometheus/prometheus

若使用docker部署直接启动镜像即可：

```bash
$ docker run --name prometheus -d -p 127.0.0.1:9090:9090 prom/prometheus:v2.28.0
```



我们想制作Prometheus的yaml文件，可以先启动容器进去看一下默认的启动命令：

```bash
$ docker run -d --name tmp -p 9090:9090 prom/prometheus:v2.28.0
$ docker exec -ti tmp sh
#/ ps aux
#/ cat /etc/prometheus/prometheus.yml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
```



本例中，使用k8s来部署，所需的资源清单如下：

```bash
# 需要准备配置文件，因此使用configmap的形式保存
$ cat prometheus.yml
# my global config
global:
  scrape_interval: 30s
  evaluation_interval: 30s
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']
    
# kubectl -n monitor create configmap prometheus-config --from-file=prometheus.yml


# pvc
$ cat pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: prometheus
  namespace: monitor
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs
  resources:
    requests:
      storage: 200Gi


# prometheus的资源文件
# 出现Prometheus数据存储权限问题，因为Prometheus内部使用nobody启动进程，挂载数据目录后权限为root，因此使用initContainer进行目录权限修复：
$ cat prometheus-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: monitor
  labels:
    app: prometheus
spec:
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      initContainers:
      - name: "change-permission-of-directory"
        image: busybox
        command: ["/bin/sh"]
        args: ["-c", "chown -R 65534:65534 /prometheus"]
        securityContext:
          privileged: true
        volumeMounts:
        - mountPath: "/etc/prometheus"
          name: config-volume
        - mountPath: "/prometheus"
          name: data
      containers:
      - image: prom/prometheus:v2.28.0
        name: prometheus
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"  # 指定tsdb数据路径
        - "--web.enable-lifecycle"  # 支持热更新，直接执行localhost:9090/-/reload立即生效
        - "--web.console.libraries=/usr/share/prometheus/console_libraries"
        - "--web.console.templates=/usr/share/prometheus/consoles"
        ports:
        - containerPort: 9090
          name: http
        volumeMounts:
        - mountPath: "/etc/prometheus"
          name: config-volume
        - mountPath: "/prometheus"
          name: data
        resources:
          requests:
            cpu: 100m
            memory: 512Mi
          limits:
            cpu: 100m
            memory: 512Mi
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: prometheus
      - configMap:
          name: prometheus-config
        name: config-volume
        
# rbac,prometheus会调用k8s api做服务发现进行抓取指标
$ cat prometheus-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitor
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  - nodes/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - "extensions"
  resources:
    - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitor


# 提供Service，为Ingress使用
$ cat prometheus-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: monitor
  labels:
    app: prometheus
spec:
  selector:
    app: prometheus
  type: ClusterIP
  ports:
    - name: web
      port: 9090
      targetPort: http

$ cat prometheus-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus
  namespace: monitor
spec:
  rules:
  - host: prometheus.luffy.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: prometheus
            port:
              number: 9090
```

部署上述资源：

```bash
# 命名空间
$ kubectl create namespace monitor

# 配置文件
$ kubectl -n monitor create configmap prometheus-config --from-file=prometheus.yml

#部署configmap
$ kubectl apply -f .

# 访问测试
$ kubectl -n monitor get ingress


```

