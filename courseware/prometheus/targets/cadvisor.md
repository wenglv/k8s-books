###### 使用cadvisor实现容器指标的采集（新）

 目前cAdvisor集成到了kubelet组件内 ，因此可以通过kubelet的接口实现容器指标的采集，具体的API为:

```bash
https://<node-ip>:10250/metrics/cadvisor    # node上的cadvisor采集到的容器指标
https://<node-ip>:10250/metrics             # node上的kubelet的指标数据

# 可以通过curl -k  -H "Authorization: Bearer xxxx" https://xxxx/xx查看
```

因此，针对容器指标来讲，我们期望的采集target是：

```bash
https://172.21.51.143:10250/metrics/cadvisor
https://172.21.51.67:10250/metrics/cadvisor
https://172.21.51.68:10250/metrics/cadvisor
```

即每个node节点都需要去采集数据，联想到prometheus的服务发现中的node类型，因此，配置：

```yaml
    - job_name: 'kubernetes-sd-cadvisor'
      kubernetes_sd_configs:
        - role: node
```

默认添加的target列表为：`__schema__://__address__ __metrics_path__`

```bash
http://172.21.51.143:10250/metrics
http://172.21.51.67:10250/metrics
http://172.21.51.68:10250/metrics
```

和期望值不同的是`__schema__`和`__metrics_path__`，针对`__metrics_path__`可以使用relabel修改：

```yaml
      relabel_configs:
      - target_label: __metrics_path__
        replacement: /metrics/cadvisor
```

针对`__schema__`：

```yaml
      - job_name: 'kubernetes-sd-cadvisor'
        kubernetes_sd_configs:
          - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - target_label: __metrics_path__
          replacement: /metrics/cadvisor
```

重新应用配置，然后重建Prometheus的pod。查看targets列表，查看cadvisor指标，比如container_cpu_system_seconds_total，container_memory_usage_bytes

综上，利用node类型的服务发现，可以实现对daemonset类型服务的目标自动发现以及监控数据抓取。

补充：

若想采集kubelet的指标：

```yaml
      - job_name: 'kubernetes-sd-kubelet'
        kubernetes_sd_configs:
          - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
```

