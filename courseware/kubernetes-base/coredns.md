###### 服务发现

在k8s集群中，组件之间可以通过定义的Service名称实现通信。

演示服务发现：

```bash
## 演示思路：在myblog的容器中直接通过service名称访问服务，观察是否可以访问通

# 先查看服务
$ kd get svc
NAME     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
myblog   ClusterIP   10.99.174.93    <none>        80/TCP     59m
mysql    ClusterIP   10.108.214.84   <none>        3306/TCP   35m

# 进入myblog容器
$ kd exec -ti myblog-5c97d79cdb-j485f bash
[root@myblog-5c97d79cdb-j485f myblog]# curl mysql:3306
5.7.29 )→  (mysql_native_password ot packets out of order
[root@myblog-5c97d79cdb-j485f myblog]# curl myblog/blog/index/
我的博客列表



```

虽然podip和clusterip都不固定，但是service name是固定的，而且具有完全的跨集群可移植性，因此组件之间调用的同时，完全可以通过service name去通信，这样避免了大量的ip维护成本，使得服务的yaml模板更加简单。因此可以对mysql和myblog的部署进行优化改造：

1. mysql可以去掉hostNetwork部署，使得服务只暴漏在k8s集群内部网络
2. configMap中数据库地址可以换成Service名称，这样跨环境的时候，配置内容基本上可以保持不用变化

修改deploy-mysql.yaml

```yaml
    spec:
      hostNetwork: true	# 去掉此行
      volumes: 
      - name: mysql-data
        hostPath: 
          path: /opt/mysql/data


        ports:
        - containerPort: 3306
          hostPort: 3306 	# 去掉此行
          protocol: TCP
```

修改configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myblog
  namespace: luffy
data:
  MYSQL_HOST: "mysql"	# 此处替换为mysql
  MYSQL_PORT: "3306"


```

应用修改：

```bash
$ kubectl delete -f deployment-mysql.yaml

## myblog不用动，会自动因健康检测不过而重启


```

服务发现实现：

 `CoreDNS`是一个`Go`语言实现的链式插件`DNS服务端`，是CNCF成员，是一个高性能、易扩展的`DNS服务端`。 

```bash
$ kubectl -n kube-system get po -o wide|grep dns
coredns-d4475785-2w4hk             1/1     Running   0          4d22h   10.244.0.64       
coredns-d4475785-s49hq             1/1     Running   0          4d22h   10.244.0.65

# 查看myblog的pod解析配置
$ kubectl -n luffy exec -ti myblog-5c97d79cdb-j485f bash
[root@myblog-5c97d79cdb-j485f myblog]# cat /etc/resolv.conf
nameserver 10.96.0.10
search luffy.svc.cluster.local svc.cluster.local cluster.local
options ndots:5

## 10.96.0.10 从哪来
$ kubectl -n kube-system get svc
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   51d

## 启动pod的时候，会把kube-dns服务的cluster-ip地址注入到pod的resolve解析配置中，同时添加对应的namespace的search域。 因此跨namespace通过service name访问的话，需要添加对应的namespace名称，
service_name.namespace
$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   26h


```

