###### Service负载均衡之NodePort

cluster-ip为虚拟地址，只能在k8s集群内部进行访问，集群外部如果访问内部服务，实现方式之一为使用NodePort方式。NodePort会默认在 30000-32767 ，不指定的会随机使用其中一个。

`myblog/deployment/svc-myblog-nodeport.yaml`

```bash
apiVersion: v1
kind: Service
metadata:
  name: myblog-np
  namespace: luffy
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8002
  selector:
    app: myblog
  type: NodePort


```

查看并访问服务：

```bash
$ kd create -f svc-myblog-nodeport.yaml
service/myblog-np created
$ kd get svc
NAME        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
myblog      ClusterIP   10.99.174.93     <none>        80/TCP         102m
myblog-np   NodePort    10.105.228.101   <none>        80:30647/TCP   4s
mysql       ClusterIP   10.108.214.84    <none>        3306/TCP       77m

#集群内每个节点的NodePort端口都会进行监听
$ curl 172.21.51.143:30647/blog/index/
我的博客列表
$ curl 172.21.51.143:30647/blog/index/
我的博客列表
## 浏览器访问


```

思考：

1. NodePort的端口监听如何转发到对应的Pod服务？

2. CLUSTER-IP为虚拟IP，集群内如何通过虚拟IP访问到具体的Pod服务？