##### 使用ingress来访问网格服务

`front-tomcat-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: front-tomcat
  namespace: istio-demo
spec:
  rules:
  - host: tomcat.istio-demo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: front-tomcat
            port:
              number: 8080
```

使用浏览器访问查看效果。

只有网格内部访问会遵从`virtualservice`的规则，在宿主机中直接访问Service的ClusterIP还是按照默认的规则转发。

Ingress：对接ingress controller，实现外部流量进入集群内部，只适用于 HTTP 流量，使用方式也很简单，只能对 service、port、HTTP 路径等有限字段匹配来路由流量，这导致它无法路由如 MySQL、Redis 和各种私有 RPC 等 TCP 流量。要想直接路由南北向的流量，只能使用 Service 的 LoadBalancer 或 NodePort，前者需要云厂商支持，后者需要进行额外的端口管理。有些 Ingress controller 支持暴露 TCP 和 UDP 服务，但是只能使用 Service 来暴露，Ingress 本身是不支持的，例如 nginx ingress controller，服务暴露的端口是通过创建 ConfigMap 的方式来配置的。

##### ingressgateway访问网格服务

![](images\gateways.svg)

对于入口流量管理，您可能会问： 为什么不直接使用 Kubernetes Ingress API ？ 原因是 Ingress API 无法表达 Istio 的路由需求。 Ingress 试图在不同的 HTTP 代理之间取一个公共的交集，因此只能支持最基本的 HTTP 路由，最终导致需要将代理的其他高级功能放入到注解（annotation）中，而注解的方式在多个代理之间是不兼容的，无法移植。

Istio `Gateway` 通过将 L4-L6 配置与 L7 配置分离的方式克服了 `Ingress` 的这些缺点。 `Gateway` 只用于配置 L4-L6 功能（例如，对外公开的端口，TLS 配置），所有主流的L7代理均以统一的方式实现了这些功能。 然后，通过在 `Gateway` 上绑定 `VirtualService` 的方式，可以使用标准的 Istio 规则来控制进入 `Gateway` 的 HTTP 和 TCP 流量。



`front-tomcat-gateway.yaml`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: front-tomcat-gateway
  namespace: istio-demo
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - tomcat.istio-demo.com
```



效果是在Istio的ingress网关上加了一条规则，允许``tomcat.istio-demo.com` 的外部http流量进入到网格中，但是只是接受访问和流量输入，当流量到达这个网关时，它还不知道发送到哪里去。

网关已准备好接收流量，我们必须告知它将收到的流量发往何处，这就用到了前面使用过的`VirtualService`。

要为进入上面的 Gateway 的流量配置相应的路由，必须为同一个 host 定义一个 `VirtualService`，并使用配置中的 `gateways` 字段绑定到前面定义的 `Gateway` 上 

 `front-tomcat-gateway-virtualservice.yaml`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: gateway-front-tomcat
  namespace: istio-demo
spec:
  gateways:
  - front-tomcat-gateway
  hosts:
  - tomcat.istio-demo.com
  http:
  - name: front-tomcat-route
    route:
    - destination:
        host: front-tomcat
        subset: v1
      weight: 90
    - destination:
        host: front-tomcat
        subset: v2
      weight: 10
```

该网关列表指定，只有通过我们指定的网关 `front-tomcat-gateway` 的流量是允许的。所有其他外部请求将被拒绝，并返回 404 响应。

>请注意，在此配置中，来自网格中其他服务的内部请求不受这些规则约束



```bash
$ kubectl apply -f front-tomcat-gateway-virtualservice.yaml
$ kubectl apply -f front-tomcat-gateway.yaml
```

模拟访问：

```bash
$ kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}'
30779
$ curl  -HHost:tomcat.istio-demo.com 172.21.51.67:30779/
```

`172.21.51.143:30779`地址从何而来？

![](images\gateway.png)



浏览器访问: `http://tomcat.istio-demo.com:30779/`

如何实现不加端口访问网格内服务？

```bash
# 在一台80端口未被占用的机器中，如k8s-slave1,ip为172.21.51.67
$ cat nginx-istio-dpl.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    name: nginx-istio
  name: nginx-istio
  namespace: istio-system
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-istio
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        name: nginx-istio
    spec:
      containers:
      - image: nginx:alpine
        imagePullPolicy: IfNotPresent
        name: nginx
        volumeMounts:
        - mountPath: /etc/nginx/conf.d
          name: nginx-istio
      hostNetwork: true
      nodeSelector:
        istio-nginx: "true"
      volumes:
      - configMap:
          defaultMode: 420
          name: nginx-istio
        name: nginx-istio
$ cat nginx-istio-configmap.yaml
apiVersion: v1
data:
  default.conf: |
    server {
        listen       80;
        listen  [::]:80;
        server_name  localhost;

        #charset koi8-r;
        #access_log  /var/log/nginx/host.access.log  main;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
    }
  tomcat.conf: |
    upstream tomcat-istiodemo {
      server 10.108.185.207:80;
    }
    server {
        listen       80;
        listen  [::]:80;
        server_name  tomcat.istio-demo.com;

        location / {
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_http_version 1.1;
            proxy_pass http://tomcat-istiodemo;
        }
    }
kind: ConfigMap
metadata:
  name: nginx-istio
  namespace: istio-system

$ nginx -s reload
```

本地配置hosts

```bash
172.21.51.55 tomcat.istio-demo.com
```

直接访问`http://tomcat.istio-demo.com` 即可实现外部域名访问到网格内部服务

