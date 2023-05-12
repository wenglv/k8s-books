##### Kubernetes服务访问之Service

通过以前的学习，我们已经能够通过Deployment来创建一组Pod来提供具有高可用性的服务。虽然每个Pod都会分配一个单独的Pod IP，然而却存在如下两个问题：

- Pod IP仅仅是集群内可见的虚拟IP，外部无法访问。
- Pod IP会随着Pod的销毁而消失，当ReplicaSet对Pod进行动态伸缩时，Pod IP可能随时随地都会变化，这样对于我们访问这个服务带来了难度。

###### Service 负载均衡之Cluster IP

service是一组pod的服务抽象，相当于一组pod的LB，负责将请求分发给对应的pod。service会为这个LB提供一个IP，一般称为cluster IP 。使用Service对象，通过selector进行标签选择，找到对应的Pod:

`myblog/deployment/svc-myblog.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myblog
  namespace: luffy
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8002
  selector:
    app: myblog
  type: ClusterIP


```

操作演示：

```bash
## 别名
$ alias kd='kubectl -n luffy'

## 创建服务
$ kd create -f svc-myblog.yaml
$ kd get po --show-labels
NAME                      READY   STATUS    RESTARTS   AGE    LABELS
myblog-5c97d79cdb-jn7km   1/1     Running   0          6m5s   app=myblog
mysql-85f4f65f99-w6jkj    1/1     Running   0          176m   app=mysql

$ kd get svc
NAME     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
myblog   ClusterIP   10.99.174.93   <none>        80/TCP    7m50s

$ kd describe svc myblog
Name:              myblog
Namespace:         demo
Labels:            <none>
Annotations:       <none>
Selector:          app=myblog
Type:              ClusterIP
IP:                10.99.174.93
Port:              <unset>  80/TCP
TargetPort:        8002/TCP
Endpoints:         10.244.0.68:8002
Session Affinity:  None
Events:            <none>

## 扩容myblog服务
$ kd scale deploy myblog --replicas=2
deployment.extensions/myblog scaled

## 再次查看
$ kd describe svc myblog
Name:              myblog
Namespace:         demo
Labels:            <none>
Annotations:       <none>
Selector:          app=myblog
Type:              ClusterIP
IP:                10.99.174.93
Port:              <unset>  80/TCP
TargetPort:        8002/TCP
Endpoints:         10.244.0.68:8002,10.244.1.158:8002
Session Affinity:  None
Events:            <none>


```

Service与Pod如何关联:

service对象创建的同时，会创建同名的endpoints对象，若服务设置了readinessProbe, 当readinessProbe检测失败时，endpoints列表中会剔除掉对应的pod_ip，这样流量就不会分发到健康检测失败的Pod中

```bash
$ kd get endpoints myblog
NAME     ENDPOINTS                            AGE
myblog   10.244.0.68:8002,10.244.1.158:8002   7m


```

Service Cluster-IP如何访问:

```bash
$ kd get svc myblog
NAME   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
myblog   ClusterIP   10.99.174.93   <none>        80/TCP    13m
$ curl 10.99.174.93/blog/index/


```

为mysql服务创建service：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: luffy
spec:
  ports:
  - port: 3306
    protocol: TCP
    targetPort: 3306
  selector:
    app: mysql
  type: ClusterIP


```

访问mysql：

```bash
$ kd get svc mysql
mysql    ClusterIP   10.108.214.84   <none>        3306/TCP   3s
$ curl 10.108.214.84:3306


```

目前使用hostNetwork部署，通过宿主机ip+port访问，弊端：

- 服务使用hostNetwork，使得宿主机的端口大量暴漏，存在安全隐患
- 容易引发端口冲突

服务均属于k8s集群，尽可能使用k8s的网络访问，因此可以对目前myblog访问mysql的方式做改造：

- 为mysql创建一个固定clusterIp的Service，把clusterIp配置在myblog的环境变量中
- 利用集群服务发现的能力，组件之间通过service name来访问

