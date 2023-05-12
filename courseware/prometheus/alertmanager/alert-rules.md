###### 配置报警规则

目前Prometheus与Alertmanager已经连通，接下来我们可以针对收集到的各类指标配置报警规则，一旦满足报警规则的设置，则Prometheus将报警信息推送给Alertmanager，进而转发到我们配置的邮件中。

在哪里配置？同样是在prometheus-configmap中：

```bash
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
    # Load rules once and periodically evaluate them according to the global  'evaluation_interval'.
    rule_files:
      - /etc/prometheus/alert_rules.yml
      # - "first_rules.yml"
      # - "second_rules.yml"
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
...
```



rules.yml我们同样使用configmap的方式挂载到prometheus容器内部，因此只需要在已有的configmap中加一个数据项目

```bash
$ vim prometheus-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitor
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    alerting:
      alertmanagers:
      - static_configs:
        - targets:
          - alertmanager:9093
    # Load rules once and periodically evaluate them according to the global  'evaluation_interval'.
    rule_files:
      - /etc/prometheus/alert_rules.yml
      # - "first_rules.yml"
      # - "second_rules.yml"
    scrape_configs:
    - job_name: 'prometheus'
      static_configs:
      - targets: ['localhost:9090']
... # 省略中间部分
  alert_rules.yml: |
    groups:
    - name: node_metrics
      rules:
      - alert: NodeLoad
        expr: node_load15 < 1
        for: 2m
        annotations:
          summary: "{{$labels.instance}}: Low node load detected"
          description: "{{$labels.instance}}: node load is below 1 (current value is: {{ $value }}"
```



告警规则的几个要素：

- `group.name`：告警分组的名称，一个组下可以配置一类告警规则，比如都是物理节点相关的告警
- `alert`：告警规则的名称
- `expr`：是用于进行报警规则 PromQL 查询语句，expr通常是布尔表达式，可以让Prometheus根据计算的指标值做 true or false 的判断
- `for`：评估等待时间（Pending Duration），用于表示只有当触发条件持续一段时间后才发送告警，在等待期间新产生的告警状态为`pending`，屏蔽掉瞬时的问题，把焦点放在真正有持续影响的问题上
- `labels`：自定义标签，允许用户指定额外的标签列表，把它们附加在告警上，可以用于后面做路由判断，通知到不同的终端，通常被用于添加告警级别的标签
- `annotations`：指定了另一组标签，它们不被当做告警实例的身份标识，它们经常用于存储一些额外的信息，用于报警信息的展示之类的

规则配置中，支持模板的方式，其中：

- {{$labels}}可以获取当前指标的所有标签，支持{{$labels.instance}}或者{{$labels.job}}这种形式

- {{ $value }}可以获取当前计算出的指标值

  

更新配置并软重启，并查看Prometheus报警规则。



一个报警信息在生命周期内有下面3种状态：

- `inactive`: 表示当前报警信息处于非活动状态，即不满足报警条件
- `pending`: 表示在设置的阈值时间范围内被激活了，即满足报警条件，但是还在观察期内
- `firing`: 表示超过设置的阈值时间被激活了，即满足报警条件，且报警触发时间超过了观察期，会发送到Alertmanager端



对于已经 `pending` 或者 `firing` 的告警，Prometheus 也会将它们存储到时间序列`ALERTS{}`中。当然我们也可以通过表达式去查询告警实例：

```bash
ALERTS{}
```



查看Alertmanager日志：

```bash
level=warn ts=2020-07-28T13:43:59.430Z caller=notify.go:674 component=dispatcher receiver=email integration=email[0] msg="Notify attempt failed, will retry later" attempts=1 err="*email.loginAuth auth: 550 User has no permission"
```

说明告警已经推送到Alertmanager端了，但是邮箱登录的时候报错，这是因为邮箱默认没有开启第三方客户端登录。因此需要登录163邮箱设置SMTP服务允许客户端登录。

