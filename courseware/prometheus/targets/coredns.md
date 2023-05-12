##### 添加监控目标

无论是业务应用还是k8s系统组件，只要提供了metrics api，并且该api返回的数据格式满足标准的Prometheus数据格式要求即可。

其实，很多组件已经为了适配Prometheus采集指标，添加了对应的/metrics api，比如

CoreDNS：

```bash
$ kubectl -n kube-system get po -owide|grep coredns
coredns-58cc8c89f4-nshx2             1/1     Running   6          22d   10.244.0.20  
coredns-58cc8c89f4-t9h2r             1/1     Running   7          22d   10.244.0.21

$ curl 10.244.0.20:9153/metrics
```

 

修改target配置：

```bash
$ kubectl -n monitor edit configmap prometheus-config
...
    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
        - targets: ['localhost:9090']
      - job_name: 'coredns'
        static_configs:
        - targets: ['10.96.0.10:9153']
      
$ kubectl apply -f prometheus-configmap.yaml

# 等待30s左右，重启Prometheus进程
$ kubectl -n monitor get po -owide
prometheus-5cd4d47557-758r5   1/1     Running   0          12m   10.244.2.104
$ curl -XPOST 10.244.2.104:9090/-/reload
```

