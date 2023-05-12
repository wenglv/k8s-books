执行的操作：

- 使用istioctl为pod注入了sidecar
- 创建了virtualservice和destinationrule

如何最终影响到了pod的访问行为？

###### 宏观角度

nginx的配置中，可以提供类似如下的配置片段实现按照权重的转发：

![](images\nginx-weight.png)

因为nginx是代理层，可以转发请求，istio也实现了流量转发的效果，肯定也有代理层，并且识别了前面创建的虚拟服务中定义的规则。



```bash
$ istioctl kube-inject -f front-tomcat-dpl-v1.yaml
```

可以看到注入后yaml中增加了很多内容：

![](images\inject.jpg)





pod被istio注入后，被纳入到服务网格中，每个pod都会添加一个名为istio-proxy的容器（常说的sidecar容器），istio-proxy容器中有两个进程，一个是`piolot-agent`，一个是`envoy`



![](images\ll-3.jpg)

```bash
$ kubectl -n istio-demo exec -ti front-tomcat-v1-78cf497978-ppwwk -c istio-proxy bash
# ps aux
```



目前已知：

- 在istio网格内，每个Pod都会被注入一个envoy代理
- envoy充当nginx的角色，做为proxy代理，负责接管pod的入口和出口流量

目前，还需要搞清楚几个问题：

- istio-init初始化容器作用是什么？
- istio-proxy如何接管业务服务的出入口流量？

