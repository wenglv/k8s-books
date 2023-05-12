##### EFK功能验证

###### 验证思路

在slave节点中启动服务，同时往标准输出中打印测试日志，到kibana中查看是否可以收集

###### 创建测试容器

`efk/test-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  nodeSelector:
    fluentd: "true"
  containers:
  - name: count
    image: alpine:3.6
    args: [/bin/sh, -c,
            'i=0; while true; do echo "$i: $(date)"; i=$((i+1)); sleep 1; done']
```

```bash
$ kubectl get po  
NAME                          READY   STATUS    RESTARTS   AGE  
counter                       1/1     Running   0          6s

```

###### 配置kibana

登录kibana界面，按照截图的顺序操作：

![](images\kibana-op1.png)

![](images\kibana-op2.png)

![](images\kibana-op3.png)

![](images\kibana-op4.png)

也可以通过其他元数据来过滤日志数据，比如可以单击任何日志条目以查看其他元数据，如容器名称，Kubernetes 节点，命名空间等，比如kubernetes.pod_name : counter

到这里，我们就在 Kubernetes 集群上成功部署了 EFK ，要了解如何使用 Kibana 进行日志数据分析，可以参考 Kibana 用户指南文档：https://www.elastic.co/guide/en/kibana/current/index.html

