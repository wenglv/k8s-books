###### Prometheus的服务发现与Relabeling

之前已经给Prometheus配置了RBAC，有读取node的权限，因此Prometheus可以去调用Kubernetes API获取node信息，所以Prometheus通过与 Kubernetes API 集成，提供了内置的服务发现分别是：`Node`、`Service`、`Pod`、`Endpoints`、`Ingress` 

配置job即可：

```bash
      - job_name: 'kubernetes-sd-node-exporter'
        kubernetes_sd_configs:
          - role: node
```

重新reload后查看效果：



![](images\prometheus-target-err1.jpg)

默认访问的地址是http://node-ip/10250/metrics，10250是kubelet API的服务端口，说明Prometheus的node类型的服务发现模式，默认是和kubelet的10250绑定的，而我们是期望使用node-exporter作为采集的指标来源，因此需要把访问的endpoint替换成http://node-ip:9100/metrics。

![](images\when-relabel-work.png)

在真正抓取数据前，Prometheus提供了relabeling的能力。怎么理解？

查看Target的Label列，可以发现，每个target对应会有很多Before Relabeling的标签，这些__开头的label是系统内部使用，不会存储到样本的数据里，但是，我们在查看数据的时候，可以发现，每个数据都有两个默认的label，即：

```bash
prometheus_notifications_dropped_total{instance="localhost:9090",job="prometheus"}	
```

instance的值其实则取自于`__address__`

这种发生在采集样本数据之前，对Target实例的标签进行重写的机制在Prometheus被称为Relabeling。 

因此，利用relabeling的能力，只需要将`__address__`替换成node_exporter的服务地址即可。

```bash
      - job_name: 'kubernetes-sd-node-exporter'
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
        - source_labels: [__address__]
          regex: '(.*):10250'
          replacement: '${1}:9100'
          target_label: __address__
          action: replace
```

再次更新Prometheus服务后，查看targets列表及node-exporter提供的指标，node_load1

