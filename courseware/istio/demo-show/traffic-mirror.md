##### 流量镜像

###### 介绍

很多情况下，当我们对服务做了重构，或者我们对项目做了重大优化时，怎么样保证服务是健壮的呢？在传统的服务里，我们只能通过大量的测试，模拟在各种情况下服务的响应情况。虽然也有手工测试、自动化测试、压力测试等一系列手段去检测它，但是测试本身就是一个样本化的行为，即使测试人员再完善它的测试样例，无法全面的表现出线上服务的一个真实流量形态 。

流量镜像的设计，让这类问题得到了最大限度的解决。流量镜像讲究的不再是使用少量样本去评估一个服务的健壮性，而是在不影响线上坏境的前提下将线上流量持续的镜像到我们的预发布坏境中去，让重构后的服务在上线之前就结结实实地接受一波真实流量的冲击与考验，让所有的风险全部暴露在上线前夕，通过不断的暴露问题，解决问题让服务在上线前夕就拥有跟线上服务一样的健壮性。由于测试坏境使用的是真实流量，所以不管从流量的多样性，真实性，还是复杂性上都将能够得以展现，同时预发布服务也将表现出其最真实的处理能力和对异常的处理能力。 



###### 实践

![](images\shadow-in-one-mesh.png)

```bash
# 准备httpbin v1
$ cat httpbin-v1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin-v1
  namespace: bookinfo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v1
  template:
    metadata:
      labels:
        app: httpbin
        version: v1
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
        
$ istioctl kube-inject -f httpbin-v1.yaml | kubectl create -f -
$ curl $(kubectl -n bookinfo get po  -l version=v1,app=httpbin -ojsonpath='{.items[0].status.podIP}')/headers
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "10.244.0.88",
    "User-Agent": "curl/7.29.0",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "777c7af4458c5b81",
    "X-B3-Traceid": "6b98ea81618deb4f777c7af4458c5b81"
  }
}

# 准备httpbin v2
$ cat httpbin-v2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin-v2
  namespace: bookinfo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v2
  template:
    metadata:
      labels:
        app: httpbin
        version: v2
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
        
$ istioctl kube-inject -f httpbin-v2.yaml | kubectl create -f -

# Service文件
$ cat httpbin-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  namespace: bookinfo
  labels:
    app: httpbin
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: httpbin

$ kubectl apply -f httpbin-svc.yaml

# 使用www.bookinfo.com/httpbin访问,因此直接修改bookinfo这个virtualservice即可
$ kubectl -n bookinfo get vs
NAME                   GATEWAYS                HOSTS               
bookinfo               [bookinfo-gateway]      [bookinfo.com]       
gateway-front-tomcat   [productpage-gateway]   [bookinfo.luffy.com]
reviews                                        [reviews]     
$ kubectl -n bookinfo edit vs bookinfo
#添加httpbin的规则
...
  - match:
    - uri:
        prefix: /httpbin
    name: httpbin-route
    rewrite:
      uri: /
    route:
    - destination:
        host: httpbin
        subset: v1
...
# 创建gateway和virtualservice,由于都是使用http请求，因此，直接
$ cat httpbin-destinationRule.yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
  namespace: bookinfo
spec:
  host: httpbin
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
$ kubectl apply -f httpbin-destinationRule.yaml

# 访问http://www.bookinfo.com/httpbin/headers，查看日志

# 为httpbin-v1添加mirror设置，mirror点为httpbin-v2
$ kubectl -n bookinfo edit vs bookinfo
...
  - match:
    - uri:
        prefix: /httpbin
    name: httpbin-route
    rewrite:
      uri: /
    route:
    - destination:
        host: httpbin
        subset: v1
    mirror:
      host: httpbin
      subset: v2
    mirror_percent: 100
...
```
