##### Grafana

可视化面板，功能齐全的度量仪表盘和图形编辑器，支持 Graphite、zabbix、InfluxDB、Prometheus、OpenTSDB、Elasticsearch 等作为数据源，比 Prometheus 自带的图表展示功能强大太多，更加灵活，有丰富的插件，功能更加强大。 

###### 安装

注意点：

- 使用最新版本的镜像 https://github.com/grafana/grafana 
- 通过环境变量设置管理员账户密码
  - GF_SECURITY_ADMIN_USER
  - GF_SECURITY_ADMIN_PASSWORD
- 通过设置securityContext的方式让grafana进程使用root启动
- 数据挂载到本地
- 配置ingress暴露访问入口

```bash
$ cat grafana-all.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: grafana
  namespace: monitor
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs
  resources:
    requests:
      storage: 200Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitor
spec:
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: grafana
      securityContext:
        runAsUser: 0
      containers:
      - name: grafana
        image: grafana/grafana:7.5.9
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
          name: grafana
        env:
        - name: GF_SECURITY_ADMIN_USER
          value: admin
        - name: GF_SECURITY_ADMIN_PASSWORD
          value: admin
        readinessProbe:
          failureThreshold: 10
          httpGet:
            path: /api/health
            port: 3000
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 30
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /api/health
            port: 3000
            scheme: HTTP
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            cpu: 150m
            memory: 512Mi
          requests:
            cpu: 150m
            memory: 512Mi
        volumeMounts:
        - mountPath: /var/lib/grafana
          name: storage
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitor
spec:
  type: ClusterIP
  ports:
    - port: 3000
  selector:
    app: grafana

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: monitor
spec:
  rules:
  - host: grafana.luffy.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 3000
```



配置数据源：

- URL：http://prometheus:9090

如何丰富Grafana监控面板：

- 导入dashboard
- 安装相应的插件
- 自定义监控面板