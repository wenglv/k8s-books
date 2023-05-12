Prometheus 是一个开源监控系统，它本身已经成为了云原生中指标监控的事实标准 。

##### k8s集群监控体系演变史

第一版本：**Cadvisor+InfluxDB+Grafana** 

只能从主机维度进行采集，没有Namespace、Pod等维度的汇聚功能



第二版本： **Heapster+InfluxDB+Grafana** 

heapster负责调用各node中的cadvisor接口，对数据进行汇总，然后导到InfluxDB ， 可以从cluster，node，pod的各个层面提供详细的资源使用情况。

![](images\monitor-earlier.png)

第三版本：Metrics-Server + Prometheus

![](images\custom-hpa.webp)



k8s对监控接口进行了标准化，主要分了三类：

- Resource Metrics

  metric-api对应的接口是 metrics.k8s.io，主要的实现就是 metrics-server，它提供的是资源的监控，比较常见的是节点级别、pod 级别、namespace 级别、class 级别。这类的监控指标都可以通过 metrics.k8s.io 这个接口获取到

- Custom Metrics

  对应的接口是 custom.metrics.k8s.io，主要的实现是 Prometheus， 它提供的是资源监控和自定义监控，资源监控和上面的资源监控其实是有覆盖关系的。

  自定义监控指的是：比如应用上面想暴露一个类似像在线人数，或者说调用后面的这个数据库的 MySQL 的慢查询。这些其实都是可以在应用层做自己的定义的，然后并通过标准的 Prometheus 的 client，暴露出相应的 metrics，然后再被 Prometheus 进行采集

- External Metrics

  对应的接口是 external.metrics.k8s.io。主要的实现厂商就是各个云厂商的 provider，通过这个 provider 可以通过云资源的监控指标



##### Prometheus架构

![](images\prometheus.svg)

- Prometheus Server ，监控、告警平台核心，抓取目标端监控数据，生成聚合数据，存储时间序列数据
- exporter，由被监控的对象提供，提供API暴漏监控对象的指标，供prometheus 抓取
  - node-exporter
  - blackbox-exporter
  - redis-exporter
  - mysql-exporter
  - custom-exporter
  - ...
- pushgateway，提供一个网关地址，外部数据可以推送到该网关，prometheus也会从该网关拉取数据
- Alertmanager，接收Prometheus发送的告警并对于告警进行一系列的处理后发送给指定的目标
- Grafana：配置数据源，图标方式展示数据 

