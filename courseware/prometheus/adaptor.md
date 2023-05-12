##### 自定义指标实现业务伸缩



###### Kubernetes Metrics API体系回顾

前面章节，我们讲过基于CPU和内存的HPA，即利用metrics-server及HPA，可以实现业务服务可以根据pod的cpu和内存进行弹性伸缩。

![](images\hpa-prometheus-custom.png)



k8s对监控接口进行了标准化：

- Resource Metrics

  对应的接口是 metrics.k8s.io，主要的实现就是 metrics-server

- Custom Metrics

  对应的接口是 custom.metrics.k8s.io，主要的实现是 Prometheus， 它提供的是资源监控和自定义监控

![](images\k8s-metrics.png)

安装完metrics-server后，利用kube-aggregator的功能，实现了metrics api的注册。可以通过如下命令

```bash
$ kubectl api-versions
...
metrics.k8s.io/v1beta1
```

HPA通过使用该API获取监控的CPU和内存资源：

```bash
# 查询nodes节点的cpu和内存数据
$ kubectl get --raw="/apis/metrics.k8s.io/v1beta1/nodes"|jq

$ kubectl get --raw="/apis/metrics.k8s.io/v1beta1/pods"|jq

# 若本机没有安装jq命令，可以参考如下方式进行安装
$ wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
$ rpm -ivh epel-release-latest-7.noarch.rpm
$ yum install -y jq
```



同样，为了实现通用指标的采集，需要部署Prometheus Adapter，来提供`custom.metrics.k8s.io`，作为HPA获取通用指标的入口。



###### Adapter安装对接

项目地址为： https://github.com/DirectXMan12/k8s-prometheus-adapter

```bash
$ git clone -b v0.8.4 https://github.com/DirectXMan12/k8s-prometheus-adapter.git
```

查看部署说明  https://github.com/DirectXMan12/k8s-prometheus-adapter/tree/v0.8.4/deploy 

1. 镜像使用官方提供的v0.8.4最新版  https://hub.docker.com/r/directxman12/k8s-prometheus-adapter/tags 

2. 准备证书

   ```bash
   $ export PURPOSE=serving
   $ openssl req -x509 -sha256 -new -nodes -days 3650 -newkey rsa:2048 -keyout ${PURPOSE}.key -out ${PURPOSE}.crt -subj "/CN=ca"
   
   $ kubectl -n monitor create secret generic cm-adapter-serving-certs --from-file=./serving.crt --from-file=./serving.key 
   
   # 查看证书
   $ kubectl -n monitor describe secret cm-adapter-serving-certs
   ```

3. 替换命名空间

   ```bash
   # 资源清单文件默认用的命名空间是custom-metrics，替换为本例中使用的monitor
   $ sed -i 's/namespace: custom-metrics/namespace: monitor/g' manifests/*
   ```

4. 配置adapter对接的Prometheus地址

   ```bash
   # 由于adapter会和Prometheus交互，因此需要配置对接的Prometheus地址
   # 替换掉28行：yamls/custom-metrics-apiserver-deployment.yaml 中的--prometheus-url
   $ vim manifests/custom-metrics-apiserver-deployment.yaml
   ...
        18     spec:
        19       serviceAccountName: custom-metrics-apiserver
        20       containers:
        21       - name: custom-metrics-apiserver
        22         image: directxman12/k8s-prometheus-adapter-amd64:v0.7.0
        23         args:
        24         - --secure-port=6443
        25         - --tls-cert-file=/var/run/serving-cert/serving.crt
        26         - --tls-private-key-file=/var/run/serving-cert/serving.key
        27         - --logtostderr=true
        28         - --prometheus-url=http://prometheus:9090/
        29         - --metrics-relist-interval=1m
        30         - --v=10
        31         - --config=/etc/adapter/config.yaml
   ...
   ```

   

5. 部署服务

   ```bash
   $ mv manifests/custom-metrics-config-map.yaml .
   $ vi manifests/custom-metrics-configmap.yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: adapter-config
     namespace: monitor
   data:
     config.yaml: |
       rules:
       - {}
   $ kubectl apply -f manifests/
   ```



验证一下：

```bash
$ kubectl api-versions|grep metrics
custom.metrics.k8s.io/v1beta1
custom.metrics.k8s.io/v1beta2
external.metrics.k8s.io/v1beta1
metrics.k8s.io/v1beta1

$ kubectl get --raw /apis/custom.metrics.k8s.io/v1beta2 |jq
{                                                    
  "kind": "APIResourceList",                         
  "apiVersion": "v1",
  "groupVersion": "custom.metrics.k8s.io/v1beta2",
  "resources": []                                    
}
```



