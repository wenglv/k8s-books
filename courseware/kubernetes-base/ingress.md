##### Kubernetes服务访问之Ingress

对于Kubernetes的Service，无论是Cluster-Ip和NodePort均是四层的负载，集群内的服务如何实现七层的负载均衡，这就需要借助于Ingress，Ingress控制器的实现方式有很多，比如nginx, Contour, Haproxy, trafik, Istio。几种常用的ingress功能对比和选型可以参考[这里](https://www.kubernetes.org.cn/5948.html)

Ingress-nginx是7层的负载均衡器 ，负责统一管理外部对k8s cluster中Service的请求。主要包含：

- ingress-nginx-controller：根据用户编写的ingress规则（创建的ingress的yaml文件），动态的去更改nginx服务的配置文件，并且reload重载使其生效（是自动化的，通过lua脚本来实现）；

- Ingress资源对象：将Nginx的配置抽象成一个Ingress对象

  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: ingress-wildcard-host
  spec:
    rules:
    - host: "foo.bar.com"
      http:
        paths:
        - pathType: Prefix
          path: "/bar"
          backend:
            service:
              name: service1
              port:
              number: 80
    - host: "bar.foo.com"
      http:
        paths:
        - pathType: Prefix
          path: "/foo"
          backend:
            service:
              name: service2
              port:
                number: 80
  ```

  

###### 示意图：

![](images/ingress.webp)

###### 实现逻辑

1）ingress controller通过和kubernetes api交互，动态的去感知集群中ingress规则变化
2）然后读取ingress规则(规则就是写明了哪个域名对应哪个service)，按照自定义的规则，生成一段nginx配置
3）再写到nginx-ingress-controller的pod里，这个Ingress controller的pod里运行着一个Nginx服务，控制器把生成的nginx配置写入/etc/nginx/nginx.conf文件中
4）然后reload一下使配置生效。以此达到域名分别配置和动态更新的问题。

###### 安装

  [官方文档](https://github.com/kubernetes/ingress-nginx/blob/master/docs/deploy/index.md)

```bash
$ wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml
## 或者使用myblog/deployment/ingress/mandatory.yaml
## 修改部署节点
$ grep -n5 nodeSelector mandatory.yaml
212-    spec:
213-      hostNetwork: true #添加为host模式
214-      # wait up to five minutes for the drain of connections
215-      terminationGracePeriodSeconds: 300
216-      serviceAccountName: nginx-ingress-serviceaccount
217:      nodeSelector:
218-        ingress: "true"		#替换此处，来决定将ingress部署在哪些机器
219-      containers:
220-        - name: nginx-ingress-controller
221-          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0
222-          args:

```

创建ingress

```bash
# 为k8s-master节点添加label
$ kubectl label node k8s-master ingress=true

$ kubectl apply -f mandatory.yaml

```



使用示例：`myblog/deployment/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myblog
  namespace: luffy
spec:
  rules:
  - host: myblog.luffy.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: myblog
            port:
              number: 80

```

ingress-nginx动态生成upstream配置：

```yaml
$ kubectl -n ingress-nginx exec -ti nginx-ingress-xxxxxxx bash
# ps aux
# cat /etc/nginx/nginx.conf|grep myblog -A10 -B1
...
        ## start server myblog.luffy.com
        server {
                server_name myblog.luffy.com ;

                listen 80  ;
                listen [::]:80  ;
                listen 443  ssl http2 ;
                listen [::]:443  ssl http2 ;

                set $proxy_upstream_name "-";

                ssl_certificate_by_lua_block {
                        certificate.call()
                }

                location / {

                        set $namespace      "luffy";
                        set $ingress_name   "myblog";
                        set $service_name   "myblog";
                        set $service_port   "80";
                        set $location_path  "/";

                        rewrite_by_lua_block {
                                lua_ingress.rewrite({
                                        force_ssl_redirect = false,
                                        ssl_redirect = true,
                                        force_no_ssl_redirect = false,
                                        use_port_in_redirects = false,
                                })
--
                                balancer.log()

                                monitor.call()

                                plugins.run()
                        }

                        port_in_redirect off;

                        set $balancer_ewma_score -1;
                        set $proxy_upstream_name "luffy-myblog-80";
                        set $proxy_host          $proxy_upstream_name;
                        set $pass_access_scheme  $scheme;

                        set $pass_server_port    $server_port;

                        set $best_http_host      $http_host;
                        set $pass_port           $pass_server_port;

                        set $proxy_alternative_upstream_name "";

--
                        proxy_next_upstream_timeout             0;
                        proxy_next_upstream_tries               3;

                        proxy_pass http://upstream_balancer;

                        proxy_redirect                          off;

                }

        }
        ## end server myblog.luffy.com
 ...


```

###### 访问

域名解析服务，将 `myblog.luffy.com`解析到ingress的地址上。ingress是支持多副本的，高可用的情况下，生产的配置是使用lb服务（内网F5设备，公网elb、slb、clb，解析到各ingress的机器，如何域名指向lb地址）

本机，添加如下hosts记录来演示效果。

```json
172.21.51.143 myblog.luffy.com
```

然后，访问 http://myblog.luffy.com/blog/index/

HTTPS访问：

```bash
#自签名证书
$ openssl req -x509 -nodes -days 2920 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=*.luffy.com/O=ingress-nginx"

# 证书信息保存到secret对象中，ingress-nginx会读取secret对象解析出证书加载到nginx配置中
$ kubectl -n luffy create secret tls tls-myblog --key tls.key --cert tls.crt



```

修改yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myblog
  namespace: luffy
spec:
  rules:
  - host: myblog.luffy.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: myblog
            port:
              number: 80
  tls:
  - hosts:
    - myblog.luffy.com
    secretName: tls-myblog
```

然后，访问 https://myblog.luffy.com/blog/index/

###### 常用注解说明

nginx端存在很多可配置的参数，通常这些参数在ingress的定义中被放在annotations中实现，如下为常用的一些：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myblog
  namespace: luffy
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "false"
    nginx.ingress.kubernetes.io/proxy-body-size: 1000m
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.org/client-max-body-size: 1000m
spec:
  rules:
  - host: myblog.luffy.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: myblog
            port:
              number: 80
  tls:
  - hosts:
    - myblog.luffy.com
    secretName: tls-myblog
```



###### 多路径转发及重写的实现

1. 多path转发示例：

   目标：

```none
myblog.luffy.com -> 172.21.51.143 -> /foo/aaa   service1:4200/foo/aaa
                                      /bar   service2:8080
                                      /		 myblog:80/

```

​	 实现：

```bash
$ cat detail.dpl.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: details
  labels:
    app: details
spec:
  replicas: 1
  selector:
    matchLabels:
      app: details
  template:
    metadata:
      labels:
        app: details
    spec:
      containers:
      - name: details
        image: docker.io/istio/examples-bookinfo-details-v1:1.16.2
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080

$ cat detail.svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: details
  labels:
    app: details
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: details
    
        
# reviews.dpl.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reviews
  labels:
    app: reviews
spec:
  replicas: 1
  selector:
    matchLabels:
      app: reviews
  template:
    metadata:
      labels:
        app: reviews
    spec:
      containers:
      - name: reviews
        image: docker.io/istio/examples-bookinfo-reviews-v3:1.16.2
        imagePullPolicy: IfNotPresent
        env:
        - name: LOG_DIR
          value: "/tmp/logs"
        ports:
        - containerPort: 9080

$ cat reviews.svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: reviews
  labels:
    app: reviews
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: reviews
```

准备Ingress文件

```yaml
# bookstore.ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bookstore
  namespace: default
spec:
  rules:
  - host: bookstore.luffy.com
    http:
      paths:
      - path: /reviews
        pathType: Prefix
        backend:
          service: 
            name: reviews
            port:
              number: 9080
      - path: /details
        pathType: Prefix
        backend:
          service: 
            name: details
            port:
              number: 9080
```



2. URL重写

   目标：

   ```
   bookstore.luffy.com -> 172.21.51.67 -> /api/reviews   -> reviews service
                                       /details   -> details service
   ```

   实现：

   ```powershell
   $ cat bookstore.reviews.ingress.yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: bookstore-reviews
     namespace: default
     annotations:
       nginx.ingress.kubernetes.io/rewrite-target: /reviews/$1
   spec:
     rules:
     - host: bookstore.luffy.com
       http:
         paths:
         - path: /api/reviews/(.*)
           pathType: Prefix
           backend:
             service: 
               name: reviews
               port:
                 number: 9080
   
   $ cat bookstore.details.ingress.yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: bookstore-details
     namespace: default
   spec:
     rules:
     - host: bookstore.luffy.com
       http:
         paths:
         - path: /details
           pathType: Prefix
           backend:
             service: 
               name: details
               port:
                 number: 9080
                 
   ```

   