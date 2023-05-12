

##### 场景三

###### 模型图

![](images\cj-3.jpg)



###### 资源清单

`front-tomcat-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: front-tomcat
  name: front-tomcat
  namespace: istio-demo
spec:
  ports:
  - name: http
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: front-tomcat
  type: ClusterIP
```

`front-tomcat-v2-dpl.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: front-tomcat
    version: v2
  name: front-tomcat-v2
  namespace: istio-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: front-tomcat
      version: v2
  template:
    metadata:
      labels:
        app: front-tomcat
        version: v2
    spec:
      containers:
      - image: consol/tomcat-7.0:latest
        name: front-tomcat
        command: ["/bin/sh", "-c", "echo 'hello tomcat version2'>/opt/tomcat/webapps/ROOT/index.html;/opt/tomcat/bin/deploy-and-run.sh;"]
```

`front-tomcat-virtualservice.yaml`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: front-tomcat
  namespace: istio-demo
spec:
  hosts:
  - front-tomcat
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
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: front-tomcat
  namespace: istio-demo
spec:
  host: front-tomcat
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```



```bash
$ kubectl apply -f front-tomcat-service.yaml
$ kubectl apply -f <(istioctl kube-inject -f front-tomcat-v2-dpl.yaml)
$ kubectl apply -f front-tomcat-virtualservice.yaml
```

 

