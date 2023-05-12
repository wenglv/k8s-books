##### Alertmanager

Alertmanager是一个独立的告警模块。

- 接收Prometheus等客户端发来的警报
- 通过分组、删除重复等处理，并将它们通过路由发送给正确的接收器；
- 告警方式可以按照不同的规则发送给不同的模块负责人。Alertmanager支持Email, Slack，等告警方式, 也可以通过webhook接入钉钉等国内IM工具。

![](images\alertmanager.png)



如果集群主机的内存使用率超过80%，且该现象持续了2分钟？想实现这样的监控告警，如何做？

从上图可得知设置警报和通知的主要步骤是：

- 安装和配置 Alertmanager

- 配置Prometheus与Alertmanager对话

- 在Prometheus中创建警报规则



###### 安装

Alertmanager，  https://github.com/prometheus/alertmanager#install 

```bash
./alertmanager --config.file=config.yml
```



alertmanager.yml配置文件格式：

```bash
$ cat config.yml
apiVersion: v1
data:
  config.yml: |
    global: 
      # 当alertmanager持续多长时间未接收到告警后标记告警状态为 resolved
      resolve_timeout: 5m
      # 配置邮件发送信息
      smtp_smarthost: 'smtp.163.com:25'
      smtp_from: 'earlene163@163.com'
      smtp_auth_username: 'earlene163@163.com'
      smtp_auth_password: 'GENLAXFHNDWNVVVL'
      smtp_require_tls: false
    # 所有报警信息进入后的根路由，用来设置报警的分发策略
    route:
      # 接收到的报警信息里面有许多alertname=NodeLoadHigh 这样的标签的报警信息将会批量被聚合到一个分组里面
      group_by: ['alertname']
      # 当一个新的报警分组被创建后，需要等待至少 group_wait 时间来初始化通知，如果在等待时间内当前group接收到了新的告警，这些告警将会合并为一个通知向receiver发送
      group_wait: 30s

      # 相同的group发送告警通知的时间间隔
      group_interval: 30s
      # 如果一个报警信息已经发送成功了，等待 repeat_interval 时间来重新发送
      repeat_interval: 1m

      # 默认的receiver：如果一个报警没有被一个route匹配，则发送给默认的接收器
      receiver: default

      # 上面所有的属性都由所有子路由继承，并且可以在每个子路由上进行覆盖。
      routes:
      - {}
    # 配置告警接收者的信息
    receivers:
    - name: 'default'
      email_configs:
      - to: '654147123@qq.com'
        send_resolved: true  # 接受告警恢复的通知
kind: ConfigMap
metadata:
  name: alertmanager
  namespace: monitor
```

主要配置的作用：

- [global](https://prometheus.io/docs/alerting/configuration/#configuration-file): 全局配置，包括报警解决后的超时时间、SMTP 相关配置、各种渠道通知的 API 地址等等。
- [route](https://prometheus.io/docs/alerting/configuration/#route): 用来设置报警的分发策略，它是一个树状结构，按照深度优先从左向右的顺序进行匹配。
- [receivers](https://prometheus.io/docs/alerting/configuration/#receiver): 配置告警消息接受者信息，例如常用的 email、wechat、slack、webhook 等消息通知方式。

当alertmanager接收到一条新的alert时，会先根据group_by为其确定一个聚合组group，然后等待group_wait时间，如果在此期间接收到同一group的其他alert，则这些alert会被合并，然后再发送（alertmanager发送消息单位是group）。此参数的作用是防止短时间内出现大量告警的情况下，接收者被告警淹没。

在该组的alert第一次被发送后，该组会进入睡眠/唤醒周期，睡眠周期将持续group_interval时间，在睡眠状态下该group不会进行任何发送告警的操作（但会插入/更新(根据fingerprint)group中的内容），睡眠结束后进入唤醒状态，然后检查是否需要发送新的alert或者重复已发送的alert(resolved类型的alert在发送完后会从group中剔除)。这就是group_interval的作用。

聚合组在每次唤醒才会检查上一次发送alert是否已经超过repeat_interval时间，如果超过则再次发送该告警。

配置文件：

```bash
$ kubectl apply -f  config.yml
```



其他资源清单文件:

```bash
$ cat alertmanager-all.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alertmanager
  namespace: monitor
  labels:
    app: alertmanager
spec:
  selector:
    matchLabels:
      app: alertmanager
  template:
    metadata:
      labels:
        app: alertmanager
    spec:
      volumes:
      - name: config
        configMap:
          name: alertmanager
      containers:
      - name: alertmanager
        image: prom/alertmanager:v0.21.0
        imagePullPolicy: IfNotPresent
        args:
        - "--config.file=/etc/alertmanager/config.yml"
        - "--log.level=debug"
        ports:
        - containerPort: 9093
          name: http
        volumeMounts:
        - mountPath: "/etc/alertmanager"
          name: config
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 100m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: alertmanager
  namespace: monitor
spec:
  type: ClusterIP
  ports:
    - port: 9093
  selector:
    app: alertmanager

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: alertmanager
  namespace: monitor
spec:
  rules:
  - host: alertmanager.luffy.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: alertmanager
            port:
              number: 9093
```

###### 配置Prometheus与Alertmanager对话

![](images\alertmanager.png)

是否告警是由Prometheus进行判断的，若有告警产生，Prometheus会将告警push到Alertmanager，因此，需要在Prometheus端配置alertmanager的地址：

```bash
    alerting:
      alertmanagers:
      - static_configs:
        - targets:
          - alertmanager:9093
```

因此，修改Prometheus的配置文件，然后重新加载pod

```bash
# 编辑prometheus-configmap.yaml配置，添加alertmanager内容
$ kuectl -n monitor edit configmap prometheus-config
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitor
data:
  prometheus.yml: |
    global:
      scrape_interval: 30s
      evaluation_interval: 30s
    alerting:
      alertmanagers:
      - static_configs:
        - targets:
          - alertmanager:9093
...
  
  
$ kubectl apply -f prometheus-configmap.yaml

# 现在已经有监控数据了，因此使用prometheus提供的reload的接口，进行服务重启

# 查看配置文件是否已经自动加载到pod中
$ kubectl -n monitor get po -o wide
prometheus-dcb499cbf-pljfn            1/1     Running   0          47h    10.244.1.167  

$ kubectl -n monitor exec -ti prometheus-dcb499cbf-pljfn cat /etc/prometheus/prometheus.yml |grep alertmanager

# 使用软加载的方式，
$ curl -X POST 10.244.1.167:9090/-/reload
```

