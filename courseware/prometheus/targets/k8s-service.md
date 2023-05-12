###### 集群Service服务的监控指标采集

比如集群中存在100个业务应用，每个业务应用都需要被Prometheus监控。

每个服务是不是都需要手动添加配置？有没有更好的方式？



```bash
      - job_name: 'kubernetes-sd-endpoints'
        kubernetes_sd_configs:
          - role: endpoints
```



reload prometheush，此使的Target列表中，`kubernetes-sd-endpoints`下出现了N多条数据，

![](images\prometheus-target-err2.png)

可以发现，实际上endpoint这个类型，目标是去抓取整个集群中所有的命名空间的Endpoint列表，然后使用默认的/metrics进行数据抓取，我们可以通过查看集群中的所有ep列表来做对比：

```bash
$ kubectl get endpoints --all-namespaces
```

但是实际上并不是每个服务都已经实现了/metrics监控的，也不是每个实现了/metrics接口的服务都需要注册到Prometheus中，因此，我们需要一种方式对需要采集的服务实现自主可控。这就需要利用relabeling中的keep功能。

![](images\when-relabel-work.png)

我们知道，relabel的作用对象是target的Before Relabling标签，比如说，假如通过如下定义:

```bash
- job_name: 'kubernetes-sd-endpoints'
  kubernetes_sd_configs:
  - role: endpoints
  relabel_configs:
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
    action: keep
    regex: true
```

那么就可以实现target的Before Relabling中若存在`__meta_kubernetes_service_annotation_prometheus_io_scrape`，且值为`true`的话，则会加入到kubernetes-sd-endpoints这个target中，否则就会被删除。

因此可以为我们期望被采集的服务，加上对应的Prometheus的label即可。

问题来了，怎么加？

查看coredns的metrics类型Before Relabling中的值，可以发现，存在如下类型的Prometheus的标签：

```bash
__meta_kubernetes_service_annotation_prometheus_io_scrape="true"
__meta_kubernetes_service_annotation_prometheus_io_port="9153"
```

![image-20210704113148694](C:\Users\liyongxin\AppData\Roaming\Typora\typora-user-images\image-20210704113148694.png)

这些内容是如何生成的呢，查看coredns对应的服务属性：

```bash
$ kubectl -n kube-system get service kube-dns -oyaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/port: "9153"
    prometheus.io/scrape: "true"
  creationTimestamp: "2021-06-06T02:39:44Z"
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: KubeDNS
  name: kube-dns
  namespace: kube-system
  ...
```

发现存在annotations声明，因此，可以联想到二者存在对应关系，Service的定义中的annotations里的特殊字符会被转换成Prometheus中的label中的下划线。

我们即可以使用如下配置，来定义服务是否要被抓取监控数据。

```bash
- job_name: 'kubernetes-sd-endpoints'
  kubernetes_sd_configs:
  - role: endpoints
  relabel_configs:
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
    action: keep
    regex: true
```

这样的话，我们只需要为服务定义上如下的声明，即可实现Prometheus自动采集数据

```bash
  annotations:
	prometheus.io/scrape: "true"
```



有些时候，我们业务应用提供监控数据的path地址并不一定是/metrics，如何实现兼容？

同样的思路，我们知道，Prometheus会默认使用Before Relabling中的`__metrics_path`作为采集路径，因此，我们再自定义一个annotation，`prometheus.io/path`

```bash
  annotations:
	prometheus.io/scrape: "true"
	prometheus.io/path: "/path/to/metrics"
```

这样，Prometheus端会自动生成如下标签：

```bash
__meta_kubernetes_service_annotation_prometheus_io_path="/path/to/metrics"
```

我们只需要在relabel_configs中用该标签的值，去重写`__metrics_path__`的值即可。因此：

```bash
- job_name: 'kubernetes-sd-endpoints'
  kubernetes_sd_configs:
  - role: endpoints
  relabel_configs:
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)
```



有些时候，业务服务的metrics是独立的端口，比如coredns，业务端口是53，监控指标采集端口是9153，这种情况，如何处理？

很自然的，我们会想到通过自定义annotation来处理，

```bash
  annotations:
	prometheus.io/scrape: "true"
	prometheus.io/path: "/path/to/metrics"
	prometheus.io/port: "9153"
```

如何去替换？

我们知道Prometheus默认使用Before Relabeling中的`__address__`进行作为服务指标采集的地址，但是该地址的格式通常是这样的

```bash
__address__="10.244.0.20:53"
__address__="10.244.0.21"
```

我们的目标是将如下两部分拼接在一起：

- 10.244.0.20
- prometheus.io/port定义的值，即`__meta_kubernetes_service_annotation_prometheus_io_port`的值

因此，需要使用正则规则取出上述两部分：

```bash
  - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
    action: replace
    target_label: __address__
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2
```

需要注意的几点：

- `__address__`中的`:<port>`有可能不存在，因此，使用`()?`的匹配方式进行
- 表达式中，三段`()`我们只需要第一和第三段，不需要中间括号部分的内容，因此使用`?:`的方式来做非获取匹配，即可以匹配内容，但是不会被记录到$1,$2这种变量中
- 多个source_labels中间默认使用`;`号分割，因此匹配的时候需要注意添加`;`号



此外，还可以将before relabeling 中的更多常用的字段取出来添加到目标的label中，比如：

```bash
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_service_name]
    action: replace
    target_label: kubernetes_service_name
  - source_labels: [__meta_kubernetes_pod_name]
    action: replace
    target_label: kubernetes_pod_name
```



因此，目前的relabel的配置如下：

```bash
      - job_name: 'kubernetes-sd-endpoints'
        kubernetes_sd_configs:
        - role: endpoints
        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2    
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          action: replace
          target_label: kubernetes_service_name
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name 
```



验证一下：

更新configmap并重启Prometheus服务，查看target列表。



