#### 快速入门

##### 场景一

###### 模型图

![](images\cj-1.jpg)

###### 资源清单

`front-tomcat-dpl-v1.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: front-tomcat
    version: v1
  name: front-tomcat-v1
  namespace: istio-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: front-tomcat
      version: v1
  template:
    metadata:
      labels:
        app: front-tomcat
        version: v1
    spec:
      containers:
      - image: consol/tomcat-7.0:latest
        name: front-tomcat
```

`bill-service-dpl-v1.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    service: bill-service
    version: v1
  name: bill-service-v1
  namespace: istio-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      service: bill-service
      version: v1
  template:
    metadata:
      labels:
        service: bill-service
        version: v1
    spec:
      containers:
      - image: nginx:alpine
        name: bill-service
        command: ["/bin/sh", "-c", "echo 'this is bill-service-v1'>/usr/share/nginx/html/index.html;nginx -g 'daemon off;'"]

```

`bill-service-svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    service: bill-service
  name: bill-service
  namespace: istio-demo
spec:
  ports:
  - name: http
    port: 9999
    protocol: TCP
    targetPort: 80
  selector:
    service: bill-service
  type: ClusterIP
```

###### 操作

```bash
$ kubectl create namespace istio-demo
$ kubectl apply -f front-tomcat-dpl-v1.yaml
$ kubectl apply -f bill-service-dpl-v1.yaml
$ kubectl apply -f bill-service-svc.yaml

$ kubectl -n istio-demo exec front-tomcat-v1-548b46d488-r7wv8 -- curl -s bill-service:9999
this is bill-service-v1
```



##### 场景二

后台账单服务更新v2版本，前期规划90%的流量访问v1版本，导入10%的流量到v2版本

###### 模型图

![](images\cj-2.jpg)

###### 资源清单

新增`bill-service-dpl-v2.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    service: bill-service
    version: v2
  name: bill-service-v2
  namespace: istio-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      service: bill-service
      version: v2
  template:
    metadata:
      labels:
        service: bill-service
        version: v2
    spec:
      containers:
      - image: nginx:alpine
        name: bill-service
        command: ["/bin/sh", "-c", "echo 'hello, this is bill-service-v2'>/usr/share/nginx/html/index.html;nginx -g 'daemon off;'"]
```

此时，访问规则会按照v1和v2的pod各50%的流量分配。

```bash
$ kubectl apply -f bill-service-dpl-v2.yaml
$ kubectl -n istio-demo exec front-tomcat-v1-548b46d488-r7wv8 --  curl -s bill-service:9999
```



![](images\cj-2-1.jpg)

###### 使用Istio

注入：

```bash
$ istioctl kube-inject -f bill-service-dpl-v1.yaml|kubectl apply -f -
$ istioctl kube-inject -f bill-service-dpl-v2.yaml|kubectl apply -f -
$ istioctl kube-inject -f front-tomcat-dpl-v1.yaml|kubectl apply -f -
```



若想实现上述需求，需要解决如下两个问题：

- 让访问账单服务的流量按照我们期望的比例，其实是一条路由规则，如何定义这个规则
- 如何区分两个版本的服务

两个新的资源类型：`VirtualService`和`DestinationRule`



`bill-service-destnation-rule.yaml`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dest-bill-service
  namespace: istio-demo
spec:
  host: bill-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```



`bill-service-virtualservice.yaml`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: vs-bill-service
  namespace: istio-demo
spec:
  hosts:
  - bill-service
  http:
  - name: bill-service-route
    route:
    - destination:
        host: bill-service
        subset: v1
      weight: 90
    - destination:
        host: bill-service
        subset: v2
      weight: 10
```



使用client验证流量分配是否生效。

```bash
$ kubectl apply -f bill-service-virtualservice.yaml
$ kubectl apply -f bill-service-destnation-rule.yaml
$ kubectl -n istio-demo exec front-tomcat-v1-78cf497978-ltxpf -c front-tomcat -- curl -s bill-service
```

