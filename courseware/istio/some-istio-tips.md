###### 一点小知识

**同一个Pod，不同的表现**

```bash
$ kubectl -n istio-demo exec -ti front-tomcat-v1-78cf497978-ppwwk -c front-tomcat bash
# curl bill-service:9999

$ kubectl -n istio-demo exec -ti front-tomcat-v1-78cf497978-ppwwk -c istio-proxy bash
# curl bill-service:9999
```

可以发现，在front-tomcat中的访问请求，是受到我们设置的 9：1的流量分配规则限制的，但是istio-proxy中的访问是不受限制的。

> istio-proxy自身，发起的往10.244.0.17的请求，使用的用户是 `uid=1337(istio-proxy)`，因此不会被`istio-init`初始化的防火墙规则拦截，可以直接走pod的网络进行通信。



**集群内的Service都相应的创建了虚拟出站监听器**

```bash
$ kubectl -n istio-demo exec -ti front-tomcat-v1-78cf497978-ppwwk -c front-tomcat bash
# curl sonarqube.jenkins:9000

$ istioctl pc l front-tomcat-v1-78cf497978-ppwwk.istio-demo --port 9000 
ADDRESS      PORT MATCH     DESTINATION
10.97.243.33 9000 App: HTTP Route: sonarqube.jenkins.svc.cluster.local:9000
10.97.243.33 9000 ALL       Cluster: outbound|9000||sonarqube.jenkins.svc.cluster.local

$ istioctl pc r front-tomcat-v1-78cf497978-ppwwk.istio-demo --name 'sonarqube.jenkins.svc.cluster.local:9000'

$ istioctl pc ep front-tomcat-v1-78cf497978-ppwwk.istio-demo --cluster 'outbound|9000||sonarqube.jenkins.svc.cluster.local'
```

virtualOutBound 15001 --> virtial listener 10.97.243.33_9000  --> route sonarqube.jenkins.svc.cluster.local:9000  --> cluster  outbound|9000||sonarqube.jenkins.svc.cluster.local --> 10.244.1.13:9000



**istio服务网格内，流量请求完全绕过了kube-proxy组件**

通过上述流程调试，我们可以得知，front-tomcat中访问bill-service:9999，流量是没有用到kube-proxy维护的宿主机中的iptables规则的。

![](images\kubernetes-vs-service-mesh.png)

验证一下：

```bash
# 停掉kube-proxy
$ kubectl -n kube-system edit daemonset kube-proxy
...
      dnsPolicy: ClusterFirst                 
      hostNetwork: true                       
      nodeSelector:                           
        beta.kubernetes.io/os: linux1    #把此处修改一个不存在的label值     
      priorityClassName: system-node-critical 
...

#清理iptables规则或者ipvs规则
$ iptables -F -t nat
$ ipvsadm -C

# 访问测试
$ kubectl -n istio-demo get  svc
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
bill-service   ClusterIP   10.111.219.247   <none>        9999/TCP   2d18h
$ curl 10.111.219.247:9999 

# 进入front-tomcat容器进行访问
$ kubectl -n istio-demo exec -ti front-tomcat-v1-78cf497978-ppwwk -c front-tomcat bash
# curl 10.111.219.247:9999 
# curl bill-service:9999 会因为dns解析失败而访问失败，手动配置namespaceserver即可

$ kubectl -n istio-demo exec -ti front-tomcat-v1-78cf497978-ppwwk -c istio-proxy bash
# curl curl 10.111.219.247:9999 
```

