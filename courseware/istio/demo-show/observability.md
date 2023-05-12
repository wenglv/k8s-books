#### 可观察性

###### 安装集成组件

https://istio.io/latest/docs/ops/integrations

1. Grafana

   ```bash
   $ kubectl apply -f samples/addons/grafana.yaml
   ```

2. Jaeger

   ```bash
   $ kubectl apply -f samples/addons/jaeger.yaml
   ```

3. Kiali  

   ```bash
   # 完善扩展组件地址：
   $ kubectl apply -f samples/addons/kiali.yaml
   ```

4. Prometheus

   ```bash
   $ kubectl apply -f samples/addons/prometheus.yaml
   ```



##### prometheus

Prometheus：

```bash
$ cat prometheus-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus
  namespace: istio-system
spec:
  rules:
  - host: prometheus.istio.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: prometheus
            port:
              number: 9090

$ kubectl apply -f prometheus-ingress.yaml
```

查看默认添加的targets列表：

![](images\prometheus-targets.jpg)

其中最核心的是`kubernetes-pods` 的监控，服务网格内的每个服务都作为一个target被监控，而且服务流量指标直接由sidecar容器来提供指标。

```bash
$ kubectl -n bookinfo get po -owide
$ curl 10.244.0.53:15020/stats/prometheus
```



对于这些监控指标采集的数据，可以在grafana中查看到。

```bash
$ cat grafana-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana
  namespace: istio-system
spec:
  rules:
  - host: grafana.istio.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: grafana
            port:
              number: 3000


$ for i in $(seq 1 10000); do curl -s -o /dev/null "http://bookinfo.luffy.com/productpage"; done
```

访问界面后，可以查看到Istio Mesh Dashboard等相关的dashboard，因为在grafana的资源文件中， 中以 ConfigMap 的形式挂载了 Istio各个组件的仪表盘 JSON 配置文件： 

```bash
$ kubectl -n istio-system get cm istio-services-grafana-dashboards
NAME                                DATA   AGE
istio-services-grafana-dashboards   3      7d1h
```

##### jaeger

```bash
$ cat jaeger-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jaeger
  namespace: istio-system
spec:
  rules:
  - host: jaeger.istio.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: tracing
            port:
              number: 80
  
$ kubectl apply -f jaeger-ingress.yaml
```



##### kiali

kiali 是一个 可观测性分析服务

```bash
$ cat kiali-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kiali
  namespace: istio-system
spec:
  rules:
  - host: kiali.istio.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: kiali
            port:
              number: 20001

$ kubectl apply -f kiali-ingress.yaml
```

集成了Prometheus、grafana、tracing、log、

> https://istio.io/latest/docs/tasks/observability/kiali/
>
> https://kiali.io/documentation/latest/configuration/authentication/

