###### 自定义webhook实现告警消息的推送

目前官方内置的第三方通知集成包括：邮件、 即时通讯软件（如Slack、Hipchat）、移动应用消息推送(如Pushover)和自动化运维工具（例如：Pagerduty、Opsgenie、Victorops）。可以在alertmanager的管理界面中查看到。

每一个receiver具有一个全局唯一的名称，并且对应一个或者多个通知方式：

```bash
name: <string>
email_configs:
  [ - <email_config>, ... ]
hipchat_configs:
  [ - <hipchat_config>, ... ]
slack_configs:
  [ - <slack_config>, ... ]
opsgenie_configs:
  [ - <opsgenie_config>, ... ]
webhook_configs:
  [ - <webhook_config>, ... ]
```





如果想实现告警消息推送给企业常用的即时聊天工具，如钉钉或者企业微信，如何配置？

Alertmanager的通知方式中还可以支持Webhook，通过这种方式开发者可以实现更多个性化的扩展支持。

```bash
# 警报接收者
receivers:
#ops 
- name: 'demo-webhook'
  webhook_configs:
  - send_resolved: true
    url: http://demo-webhook/alert/send
```



当我们配置了上述webhook地址，则当告警路由到`alertmanager`时，alertmanager端会向webhook地址推送POST请求：

```bash
$ curl -X POST -d"$demoAlerts"  http://demo-webhook/alert/send
$ echo $demoAlerts
{
  "version": "4",
  "groupKey": <string>, alerts (e.g. to deduplicate) ,
  "status": "<resolved|firing>", 
  "receiver": <string>, 
  "groupLabels": <object>, 
  "commonLabels": <object>, 
  "commonAnnotations": <object>, 
  "externalURL": <string>, // backlink to the Alertmanager. 
  "alerts": 
   [{ 
     "labels": <object>, 
      "annotations": <object>, 
      "startsAt": "<rfc3339>", 
      "endsAt": "<rfc3339>" 
   }] 
} 
```

因此，假如我们想把报警消息自动推送到钉钉群聊，只需要：

- 实现一个webhook，部署到k8s集群
  - 接收POST请求，将Alertmanager传过来的数据做解析，调用dingtalk的API，实现消息推送
- 配置alertmanager的receiver为webhook地址



如何给钉钉群聊发送消息？ 钉钉机器人

钉钉群聊机器人设置：

每个群聊机器人在创建的时候都会生成唯一的一个访问地址：

```bash
https://oapi.dingtalk.com/robot/send?access_token=f628f749a7ad70e86ca7bcb68658d0ce5af7c201ce8ce32acaece4c592364ca9
```

这样，我们就可以使用如下方式来模拟给群聊机器人发送请求，实现消息的推送：

```bash
curl 'https://oapi.dingtalk.com/robot/send?access_token=f628f749a7ad70e86ca7bcb68658d0ce5af7c201ce8ce32acaece4c592364ca9' \
   -H 'Content-Type: application/json' \
   -d '{"msgtype": "text","text": {"content": "我就是我, 是不一样的烟火"}}'
```





 https://gitee.com/agagin/prometheus-webhook-dingtalk 

镜像地址：timonwong/prometheus-webhook-dingtalk:master

二进制运行：

```bash
$ ./prometheus-webhook-dingtalk --config.file=config.yml
```



假如使用如下配置：

```bash
targets:
  webhook_dev:
    url: https://oapi.dingtalk.com/robot/send?access_token=f33c539fa1012e0b3500f04ea98fb89468829ed324699d67ecd2f177a1dcc0c2
  webhook_ops:
    url: https://oapi.dingtalk.com/robot/send?access_token=4778abd23dbdbaf66fc6f413e6ab9c0103a039b0054201344a22a5692cdcc54e
```

则prometheus-webhook-dingtalk启动后会自动支持如下API的POST访问：

```bash
http://locahost:8060/dingtalk/webhook_dev/send
http://localhost:8060/dingtalk/webhook_ops/send
```

这样可以使用一个prometheus-webhook-dingtalk来实现多个钉钉群的webhook地址



部署prometheus-webhook-dingtalk，从Dockerfile可以得知需要注意的点：

- 默认使用配置文件` /etc/prometheus-webhook-dingtalk/config.yml `，可以通过configmap挂载
- 该目录下还有模板文件，因此需要使用subpath的方式挂载
- 部署Service，作为Alertmanager的默认访问，服务端口默认8060



配置文件：

```bash
$ cat webhook-dingtalk-configmap.yaml
apiVersion: v1
data:
  config.yml: |
    targets:
      webhook_dev:
        url: https://oapi.dingtalk.com/robot/send?access_token=f33c539fa1012e0b3500f04ea98fb89468829ed324699d67ecd2f177a1dcc0c2
      webhook_ops:
        url: https://oapi.dingtalk.com/robot/send?access_token=4778abd23dbdbaf66fc6f413e6ab9c0103a039b0054201344a22a5692cdcc54e
kind: ConfigMap
metadata:
  name: webhook-dingtalk-config
  namespace: monitor
```

Deployment和Service

```bash
$ cat webhook-dingtalk-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webhook-dingtalk
  namespace: monitor
spec:
  selector:
    matchLabels:
      app: webhook-dingtalk
  template:
    metadata:
      labels:
        app: webhook-dingtalk
    spec:
      containers:
      - name: webhook-dingtalk
        image: timonwong/prometheus-webhook-dingtalk:master
        args:
        - "--config.file=/etc/prometheus-webhook-dingtalk/config.yml"
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - mountPath: "/etc/prometheus-webhook-dingtalk/config.yml"
          name: config
          subPath: config.yml
        ports:
        - containerPort: 8060
          name: http
        resources:
          requests:
            cpu: 50m
            memory: 100Mi
          limits:
            cpu: 50m
            memory: 100Mi
      volumes:
      - name: config
        configMap:
          name: webhook-dingtalk-config
          items:
          - key: config.yml
            path: config.yml
---
apiVersion: v1
kind: Service
metadata:
  name: webhook-dingtalk
  namespace: monitor
spec:
  selector:
    app: webhook-dingtalk
  ports:
  - name: hook
    port: 8060
    targetPort: http
```



创建：

```bash
$ kubectl apply -f webhook-dingtalk-configmap.yaml
$ kubectl apply -f webhook-dingtalk-deploy.yaml

# 查看日志，可以得知当前的可用webhook日志
$ kubectl -n monitor logs -f webhook-dingtalk-f7f5589c9-qglkd
...
file=/etc/prometheus-webhook-dingtalk/config.yml msg="Completed loading of configuration file"
level=info ts=2020-07-30T14:05:40.963Z caller=main.go:117 component=configuration msg="Loading templates" templates=
ts=2020-07-30T14:05:40.963Z caller=main.go:133 component=configuration msg="Webhook urls for prometheus alertmanager" urls="http://localhost:8060/dingtalk/webhook_dev/send http://localhost:8060/dingtalk/webhook_ops/send"
level=info ts=2020-07-30T14:05:40.963Z caller=web.go:210 component=web msg="Start listening for connections" address=:8060
```



修改Alertmanager路由及webhook配置：

```bash
$ kubectl -n monitor edit configmap alertmanager
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager
  namespace: monitor
data:
  config.yml: |-
    global:
      # 当alertmanager持续多长时间未接收到告警后标记告警状态为 resolved
      resolve_timeout: 5m
      # 配置邮件发送信息
      smtp_smarthost: 'smtp.163.com:25'
      smtp_from: 'earlene163@163.com'
      smtp_auth_username: 'earlene163@163.com'
      # 注意这里不是邮箱密码，是邮箱开启第三方客户端登录后的授权码
      smtp_auth_password: 'GXIWNXKMMEVMNHAJ'
      smtp_require_tls: false
    # 所有报警信息进入后的根路由，用来设置报警的分发策略
    route:
      # 按照告警名称分组
      group_by: ['alertname']
      # 当一个新的报警分组被创建后，需要等待至少 group_wait 时间来初始化通知，这种方式可以确保您能有足够的时间为同一分组来获取多个警报，然后一起触发这个报警信息。
      group_wait: 30s

      # 相同的group之间发送告警通知的时间间隔
      group_interval: 30s

      # 如果一个报警信息已经发送成功了，等待 repeat_interval 时间来重新发送他们，不同类型告警发送频率需要具体配置
      repeat_interval: 1m

      # 默认的receiver：如果一个报警没有被一个route匹配，则发送给默认的接收器
      receiver: default

      # 路由树，默认继承global中的配置，并且可以在每个子路由上进行覆盖。
      routes:
      - {}
    receivers:
    - name: 'default'
      email_configs:
      - to: '654147123@qq.com'
        send_resolved: true  # 接受告警恢复的通知
      webhook_configs:
      - send_resolved: true
        url: http://webhook-dingtalk:8060/dingtalk/webhook_dev/send
```

验证钉钉消息是否正常收到。



