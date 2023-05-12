##### 理解时间序列数据库（TSDB）

 ```bash
# http://localhost:9090/metrics
$ kubectl -n monitor get po -o wide
prometheus-dcb499cbf-fxttx   1/1     Running   0          13h   10.244.1.132   k8s-slave1 

$ curl http://10.244.1.132:9090/metrics
...
# HELP promhttp_metric_handler_requests_total Total number of scrapes by HTTP status code.
# TYPE promhttp_metric_handler_requests_total counter
promhttp_metric_handler_requests_total{code="200"} 5
promhttp_metric_handler_requests_total{code="500"} 0
promhttp_metric_handler_requests_total{code="503"} 0

 ```

tsdb（Time Series Database）

其中#号开头的两行分别为：

- HELP开头说明该行为指标的帮助信息，通常解释指标的含义
- TYPE开头是指明了指标的类型
  - counter 计数器
  - guage 测量器
  - histogram 柱状图
  - summary 采样点分位图统计 

其中非#开头的每一行表示当前采集到的一个监控样本：

- promhttp_metric_handler_requests_total表明了当前指标的名称
- 大括号中的标签则反映了当前样本的一些特征和维度
- 浮点数则是该监控样本的具体值。 

每次采集到的数据都会被Prometheus以time-series（时间序列）的方式保存到内存中，定期刷新到硬盘。如下所示，可以将time-series理解为一个以时间为X轴的数字矩阵： 

```bash
  ^
  │   . . . . . . . . . . . . . . . . .   . .   node_cpu{cpu="cpu0",mode="idle"}
  │     . . . . . . . . . . . . . . . . . . .   node_cpu{cpu="cpu0",mode="system"}
  │     . . . . . . . . . .   . . . . . . . .   node_load1{}
  │     . . . . . . . . . . . . . . . .   . .  
  v
    <------------------ 时间 ---------------->
```

在time-series中的每一个点称为一个样本（sample），样本由以下三部分组成：

- 指标(metric)：metric name和描述当前样本特征的labelsets;
- 时间戳(timestamp)：一个精确到毫秒的时间戳;
- 样本值(value)： 一个float64的浮点型数据表示当前样本的值。



在形式上，所有的指标(Metric)都通过如下格式标示：

```
<metric name>{<label name>=<label value>, ...}
```

- 指标的名称(metric name)可以反映被监控样本的含义（比如，`http_request_total` - 表示当前系统接收到的HTTP请求总量）。
- 标签(label)反映了当前样本的特征维度，通过这些维度Prometheus可以对样本数据进行过滤，聚合等。  

Prometheus：定期去Tragets列表拉取监控数据，存储到TSDB中，并且提供指标查询、分析的语句和接口。

