##### 流量路由

实现ingress解决不了的按照比例分配流量

###### ingress-gateway访问productpage

`productpage-gateway.yaml`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: productpage-gateway
  namespace: bookinfo
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - bookinfo.luffy.com
```

`productpage-virtualservice.yaml`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: vs-bookinfo
  namespace: bookinfo
spec:
  gateways:
  - productpage-gateway
  hosts:
  - bookinfo.luffy.com
  http:
  - route:
    - destination:
        host: productpage
        port:
          number: 9080
```



配置nginx，使用域名80端口访问。

```bash
  bookinfo-productpage.conf: |
    upstream bookinfo-productpage {
      server 10.108.185.207:80;
    }
    server {
        listen       80;
        listen  [::]:80;
        server_name  bookinfo.luffy.com;

        location / {
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_http_version 1.1;
            proxy_pass http://bookinfo-productpage;
        }
    }
```



![](images\withistio.svg)

###### 权重路由

只想访问`reviews-v3`

```bash
$ cat virtual-service-reviews-v3.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: bookinfo
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v3

$ cat destination-rule-reviews.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
  namespace: bookinfo
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3

$ kubectl apply -f virtual-service-reviews-v3.yaml

# 访问productpage测试
```



实现如下流量分配：

```bash
0% -> reivews-v1
10% -> reviews-v2
90%  -> reviews-v3
```

```bash
$ cat virtual-service-reviews-90-10.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: bookinfo
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
      weight: 10
    - destination:
        host: reviews
        subset: v3
      weight: 90

$ kubectl apply -f virtual-service-reviews-90-10.yaml

# 假如v2版本的副本数扩容为3，v2版本的流量会如何分配？  会不会变成30%？
$ kubectl -n bookinfo scale deploy reviews-v2 --replicas=3
#  istioctl pc route productpage-v1-667f4495b5-kb5qv.bookinfo --name 9080 -ojson
```



###### 访问路径路由

实现效果如下：

![](images\ll-4.jpg)

```bash
# 修改外部流量进入网格后的规则

$ cat virtualservice-bookinfo-with-uri-path.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: vs-bookinfo
  namespace: bookinfo
spec:
  gateways:
  - productpage-gateway
  hosts:
  - bookinfo.luffy.com
  http:
  - name: productpage-route
    match:
    - uri:
        prefix: /productpage
    route:
    - destination:
        host: productpage
  - name: reviews-route
    match:
    - uri:
        prefix: /reviews
    route:
    - destination:
        host: reviews
  - name: ratings-route
    match:
    - uri:
        prefix: /ratings
    route:
    - destination:
        host: ratings
  - route:
    - destination:
        host: productpage
        port:
          number: 9080
```


访问：

`http://bookinfo.com/productpage`

`http://bookinfo.com/ratings/1`

实际的访问对应为：

```bash
www.bookinfo.com/abc  -> productpage:8090/abc
www.bookinfo.com/ratings  ->  ratings:9080/ratings
www.bookinfo.com/reviews  ->  reviews:9080/reviews
```


`virtualservice`的配置中并未指定service的port端口，转发同样可以生效？

> 注意，若service中只有一个端口，则不用显式指定端口号，会自动转发到该端口中



###### 路径重写

如果想实现rewrite的功能，

```bash
www.bookinfo.com/rate  -> ratings:8090/ratings
```

```yaml
...
  - name: ratings-route
    match:
    - uri:
        prefix: /rate
    rewrite:
      uri: "/ratings"
    route:
    - destination:
        host: ratings
...
```




###### DestinationRule 转发策略

默认会使用轮询策略，此外也支持如下负载均衡模型，可以在 `DestinationRule` 中使用这些模型，将请求分发到特定的服务或服务子集。

| Name          | Description                                                  |
| ------------- | ------------------------------------------------------------ |
| `ROUND_ROBIN` | Round Robin policy. Default                                  |
| `LEAST_CONN`  | The least request load balancer uses an O(1) algorithm which selects two random healthy hosts and picks the host which has fewer active requests. |
| `RANDOM`      | The random load balancer selects a random healthy host. The random load balancer generally performs better than round robin if no health checking policy is configured. |
| `PASSTHROUGH` | This option will forward the connection to the original IP address requested by the caller without doing any form of load balancing. This option must be used with care. It is meant for advanced use cases. Refer to Original Destination load balancer in Envoy for further details. |

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:     #默认的负载均衡策略模型为随机
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1  #subset1，将流量转发到具有标签 version:v1 的 deployment 对应的服务上
    labels:
      version: v1
  - name: v2  #subset2，将流量转发到具有标签 version:v2 的 deployment 对应的服务上,指定负载均衡为轮询
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3   #subset3，将流量转发到具有标签 version:v3 的 deployment 对应的服务上
    labels:
      version: v3
```

> https://preliminary.istio.io/latest/zh/docs/reference/config/networking/destination-rule/#DestinationRule

###### 使用https

方式一：把证书绑定在外部的nginx中， nginx 443端口监听外网域名并转发请求到Istio Ingress网关IP+http端口 ，如果使用公有云lb的话（如slb，clb），可以在lb层绑定证书

方式二：在istio侧使用证书

https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/



###### header头路由

```bash
$ cat virtual-service-reviews-header.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
  namespace: bookinfo
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: luffy
    route:
    - destination:
        host: reviews
        subset: v3
  - route:
    - destination:
        host: reviews
        subset: v2
```

```bash
$ kubectl apply -f virtual-service-reviews-header.yaml

# https://github.com/nocalhost/bookinfo-productpage/blob/main/productpage.py
# 刷新观察http://bookinfo.com/productpage
```



更多支持的匹配类型可以在此处查看。

https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPMatchRequest

