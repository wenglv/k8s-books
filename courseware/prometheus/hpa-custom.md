###### 配置HPA实现自定义指标的业务伸缩

```bash
$ cat hpa-custom-metrics.yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: custom-metrics-demo
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: custom-metrics-demo
  minReplicas: 1
  maxReplicas: 3
  metrics:
  - type: Pods
    pods:
      metricName: http_requests_per_second
      targetAverageValue: 10
      
$ kubectl apply -f hpa-custom-metrics.yaml

$ kubectl get hpa
```



注意metricName为自定义的指标名称。

使用ab命令压测`front-app`服务，观察hpa的变化：

```bash
$ kubectl get svc -o wide
custom-metrics-demo   ClusterIP   10.104.110.245   <none>        80/TCP    16h

$ ab -n1000 -c 5 http://10.104.110.245:80/
```



观察hpa变化:

```bash
$ kubectl describe hpa nginx-custom-hpa
```



查看adapter日志：

```bash
$ kubectl -n monitor logs --tail=100 -f custom-metrics-apiserver-c689ff947-m5vlr
...
I0802 04:43:58.404559       1 httplog.go:90] GET /apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/%2A/nginx_vts_server_requests_per_second?labelSelector=app%3Dcustom-metrics-demo: (20.713209ms) 200 [kube-controller-manager/v1.16.0 (linux/amd64) kubernetes/2bd9643/system:serviceaccount:kube-system:horizontal-pod-autoscaler 172.21.51.143:60626]

```



实际的请求：

```bash
http://prometheus:9090/api/v1/query?query=%28sum%28rate%28nginx_vts_server_requests_per_second%7Bkubernetes_namespace%3D%22default%22%2Ckubernetes_pod_name%3D~%22custom-metrics-demo-95b5bc949-9vd8q%7Ccustom-metrics-demo-95b5bc949-qrpnp%22%2Cjob%3D%22kubernetes-sd-endpoints%22%7D%5B1m%5D%29%29+by+%28kubernetes_pod_name

I1028 08:56:05.289421       1 api.go:74] GET http://prometheus:9090/api/v1/query?query=%28sum%28rate%28nginx_vts_server_requests_total%7Bkubernetes_namespace%3D%22default%22%2Ckubernetes_pod_name%3D~%22custom-metrics-demo-95b5bc949-9vd8q%7Ccustom-metrics-demo-95b5bc949-qrpnp%22%7D%5B1m%5D%29%29+by+%28kubernetes_pod_name%29%29&time=1603875365.284 200 OK
```





补充：coredns通用指标的hpa

`coredns_dns_requests_total`指标记录了coredns域名解析记录的信息，我们想根据coredns被访问的频率来实现动态伸缩coredns服务的副本数，因此，基于上述知识，可以添加如下内容：

```yaml
$ cat custom-metrics-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
  namespace: monitor
data:
  config.yaml: |
    rules:
    - seriesQuery: 'http_requests_total'
      seriesFilters: []
      resources:
        overrides:
          kubernetes_namespace: {resource: "namespace"}
          kubernetes_pod_name: {resource: "pod"}
      name:
        as: "http_requests_per_second"
      metricsQuery: (sum(rate(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>))
    - seriesQuery: 'coredns_dns_requests_total{}'
      seriesFilters: []
      resources:
        overrides:
          kubernetes_namespace: {resource: "namespace"}
          kubernetes_pod_name: {resource: "pod"}
      name:
        as: "coredns_dns_request_count_per_second"
      metricsQuery: (sum(rate(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>))
```



coredns的hpa文件：

```yaml
$ cat coredns-hpa.yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: coredns-hpa
  namespace: kube-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: coredns
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metricName: coredns_dns_request_count_per_second
      targetAverageValue: 1000
```



可以通过正则匹配的方式，实现对于同一类别的业务指标的定义：

```bash
$ cat custom-metrics-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
  namespace: monitor
data:
  config.yaml: |
    rules:
    - seriesQuery: '{__name__=~".*_requests_total",kubernetes_pod_name!="",kubernetes_namespace!=""}'
      seriesFilters: []
      resources:
        overrides:
          kubernetes_namespace:
            resource: namespace
          kubernetes_pod_name:
            resource: pod
      name:
        matches: ^(.*)_total$
        as: "${1}_per_second"
      metricsQuery: sum(rate(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>)
```

 coredns_dns_requests_per_second

 http_requests__per_second 

再次查看当前可用的自定义指标：

```bash
$ kubectl get --raw /apis/custom.metrics.k8s.io/v1beta2 |jq
```

