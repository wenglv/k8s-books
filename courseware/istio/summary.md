#### 展望未来

Kubernets已经成为了容器调度编排的事实标准，而容器正好可以作为微服务的最小工作单元，从而发挥微服务架构的最大优势。所以未来微服务架构会围绕Kubernetes展开。而Istio和Conduit这类Service Mesh天生就是为了Kubernetes设计，它们的出现补足了Kubernetes在微服务间服务通讯上的短板。虽然Dubbo、Spring Cloud等都是成熟的微服务框架，但是它们或多或少都会和具体语言或应用场景绑定，并只解决了微服务Dev层面的问题。若想解决Ops问题，它们还需和诸如Cloud Foundry、Mesos、Docker Swarm或Kubernetes这类资源调度框架做结合：

![](images\kubernetes与springcloud.jpg)

但是这种结合又由于初始设计和生态，有很多适用性问题需要解决。

Kubernetes则不同，它本身就是一个和开发语言无关的、通用的容器管理平台，它可以支持运行云原生和传统的容器化应用。并且它覆盖了微服务的Dev和Ops阶段，结合Service Mesh，它可以为用户提供完整端到端的微服务体验。

所以我认为，未来的微服务架构和技术栈可能是如下形式：

![](images\microservice-future.jpg)

多云平台为微服务提供了资源能力（计算、存储和网络等），容器作为最小工作单元被Kubernetes调度和编排，Service Mesh管理微服务的服务通信，最后通过API Gateway向外暴露微服务的业务接口。

未来随着以Kubernetes和Service Mesh为标准的微服务框架的盛行，将大大降低微服务实施的成本，最终为微服务落地以及大规模使用提供坚实的基础和保障。





#### 小结

- 第一代为Spring Cloud为代表的服务治理能力，是和业务代码紧耦合的，没法跨编程语言去使用

- 为了可以实现通用的服务治理能力，istio会为每个业务pod注入一个sidecar代理容器

  ![](images\20200503110532.png)

- 为了能够做到服务治理，需要接管pod内的出入流量，因此通过注入的时候引入初始化容器istio-init实现pod内防火墙规则的初始化，分别将出入站流量拦截到pod内的15001和15006端口

- 同时，注入了istio-proxy容器，利用envoy代理，监听了15001和15006端口，对流量进行处理

  ![](images\envoy-config-init.png)

- istio在istio-system命名空间启动了istiod服务，用于监听用户写入etcd中的流量规则，转换成envoy可度的配置片段，通过envoy支持的xDS协议，同步到网格内的各envoy中

- envoy获取规则后，做reload，直接应用到了用户期望的转发行为

- envoy提供了强大的流量处理规则，包含了流量路由、镜像、重试、熔断、故障注入等，同时，也内置了分布式追踪、Prometheus监控的实现，业务应用对于这一切都是感知不到的





最后，通过分析bookinfo中，从 Productpage服务调用Reviews服务的 请求流程 来回顾istio重点：

![](images\envoy-traffic-route.png)

1. Productpage发起对Reviews服务的调用：`http://reviews:9080/reviews/0` 

2. 请求被Productpage Pod的iptable规则拦截，重定向到本地的15001端口 

   ```bash
   # OUTPUT 链：将所有出站数据包跳转到 ISTIO_OUTPUT 链上。
   Chain OUTPUT (policy ACCEPT 46 packets, 3926 bytes)
    pkts bytes target     prot opt in     out     source               destination
       8   480 ISTIO_OUTPUT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0
   
   # ISTIO_OUTPUT 链：选择需要重定向到 Envoy（即本地） 的出站流量，所有非 localhost 的流量全部转发到 ISTIO_REDIRECT。为了避免流量在该 Pod 中无限循环，所有到 istio-proxy 用户空间的流量都返回到它的调用点中的下一条规则，本例中即 OUTPUT 链，因为跳出 ISTIO_OUTPUT 规则之后就进入下一条链 POSTROUTING。如果目的地非 localhost 就跳转到 ISTIO_REDIRECT；如果流量是来自 istio-proxy 用户空间的，那么就跳出该链，返回它的调用链继续执行下一条规则（OUTPUT 的下一条规则，无需对流量进行处理）；所有的非 istio-proxy 用户空间的目的地是 localhost 的流量就跳转到 ISTIO_REDIRECT。
   Chain ISTIO_OUTPUT (1 references)
    pkts bytes target     prot opt in     out     source               destination
       0     0 RETURN     all  --  *      lo      127.0.0.6            0.0.0.0/0
       0     0 ISTIO_IN_REDIRECT  all  --  *      lo      0.0.0.0/0           !127.0.0.1            owner UID match 1337
       0     0 RETURN     all  --  *      lo      0.0.0.0/0            0.0.0.0/0            ! owner UID match 1337
       8   480 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner UID match 1337
       0     0 ISTIO_IN_REDIRECT  all  --  *      lo      0.0.0.0/0           !127.0.0.1            owner GID match 1337
       0     0 RETURN     all  --  *      lo      0.0.0.0/0            0.0.0.0/0            ! owner GID match 1337
       0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            owner GID match 1337
       0     0 RETURN     all  --  *      *       0.0.0.0/0            127.0.0.1
       0     0 ISTIO_REDIRECT  all  --  *      *       0.0.0.0/0            0.0.0.0/0
   
   # ISTIO_REDIRECT 链：将所有流量重定向到 Sidecar（即本地） 的 15001 端口。
   Chain ISTIO_REDIRECT (1 references)
    pkts bytes target     prot opt in     out     source               destination
       0     0 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            redir ports 15001
   ```

3. 15001端口上监听的Envoy Virtual Outbound Listener收到了该请求 

   ```bash
   $ istioctl pc listener productpage-v1-785b4dbc96-2cw5v.bookinfo --port 15001 -ojson|more
   [
       {
           "name": "virtualOutbound",
           "address": {
               "socketAddress": {
                   "address": "0.0.0.0",
                   "portValue": 15001
               }
           },
   ...
   ```

4. 请求被Virtual Outbound Listener根据原目标IP（通配）和端口（9080）转发到0.0.0.0_9080这个 outbound listener 

   ```bash
   $ istioctl pc listener productpage-v1-785b4dbc96-2cw5v.bookinfo --port 9080 -ojson
   [
       {
           "name": "0.0.0.0_9080",
           "address": {
               "socketAddress": {
                   "address": "0.0.0.0",
                   "portValue": 9080
               }
           },
           "filterChains": [
               {
                   "filterChainMatch": {
                       "applicationProtocols": [
                           "http/1.0",
                           "http/1.1",
                           "h2c"
                       ]
                   },
                   "filters": [
                       {
                           "name": "envoy.filters.network.http_connection_manager",
                           "typedConfig": {
                               "statPrefix": "outbound_0.0.0.0_9080",
                               "rds": {
                                   "configSource": {
                                       "ads": {},
                                       "resourceApiVersion": "V3"
                                   },
                                   "routeConfigName": "9080"
                               },
   ```

5. 根据0.0.0.0_9080 listener的http_connection_manager filter配置,该请求采用“9080” route进行分发

6. 9080的route中，根据domains进行匹配，将请求交给 `outbound|9080|v2|reviews.bookinfo.svc.cluster.local`这个cluster处理

   ```bash
   $ istioctl pc route productpage-v1-785b4dbc96-2cw5v.bookinfo --name 9080 -ojson
   ...
               {
                   "name": "reviews.bookinfo.svc.cluster.local:9080",
                   "domains": [
                       "reviews.bookinfo.svc.cluster.local",
                       "reviews.bookinfo.svc.cluster.local:9080",
                       "reviews",
                       "reviews:9080",
                       "reviews.bookinfo.svc.cluster",
                       "reviews.bookinfo.svc.cluster:9080",
                       "reviews.bookinfo.svc",
                       "reviews.bookinfo.svc:9080",
                       "reviews.bookinfo",
                       "reviews.bookinfo:9080",
                       "10.109.133.236",
                       "10.109.133.236:9080"
                   ],
                   "routes": [
                       {
                           "match": {
                               "prefix": "/",
                               "caseSensitive": true,
                               "headers": [
                                   {
                                       "name": "end-user",
                                       "exactMatch": "luffy"
                                   }
                               ]
                           },
                           "route": {
                               "cluster": "outbound|9080|v2|reviews.bookinfo.svc.cluster.local",
                               "timeout": "1s",
   ```

7. 该cluster为EDS

   ```bash
   $ istioctl pc cluster productpage-v1-785b4dbc96-2cw5v.bookinfo --fqdn reviews.bookinfo.svc.cluster.local --direction outbound -ojson
   ...
           "name": "outbound|9080|v3|reviews.bookinfo.svc.cluster.local",
           "type": "EDS",
           "edsClusterConfig": {
               "edsConfig": {
                   "ads": {},
                   "resourceApiVersion": "V3"
               },
               "serviceName": "outbound|9080|v3|reviews.bookinfo.svc.cluster.local"
           },
           "connectTimeout": "10s",
           "lbPolicy": "RANDOM",
           "circuitBreakers": {
               "thresholds": [
                   {
                       "maxConnections": 4294967295,
                       "maxPendingRequests": 4294967295,
                       "maxRequests": 4294967295,
                       "maxRetries": 4294967295
                   }
               ]
           },
   ```

8. 查寻EDS对应的endpoint列表

   ```bash
   $ istioctl pc endpoint productpage-v1-785b4dbc96-2cw5v.bookinfo --cluster "outbound|9080|v3|reviews.bookinfo.svc.cluster.local" -ojson
   [
       {
           "name": "outbound|9080|v3|reviews.bookinfo.svc.cluster.local",
           "addedViaApi": true,
           "hostStatuses": [
               {
                   "address": {
                       "socketAddress": {
                           "address": "10.244.0.37",
                           "portValue": 9080
                       }
                   },
   ...
   ```

9. envoy进程得到了最终需要访问的地址（reviews-v3的podip：port），由envoy做proxy转发出去

10. 此时，虽然还是会走一遍envoy的防火墙规则，但是由于是1337用户发起的请求，因此不会被再次拦截，直接走kubernetes的集群网络发出去

11. 请求到达reviews-v3， 被iptable规则拦截，重定向到本地的15006端口 

    ```bash
    # PREROUTING 链：用于目标地址转换（DNAT），将所有入站 TCP 流量跳转到 ISTIO_INBOUND 链上。
    Chain PREROUTING (policy ACCEPT 148 packets, 8880 bytes)
     pkts bytes target     prot opt in     out     source               destination
      148  8880 ISTIO_INBOUND  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0
    
    # INPUT 链：处理输入数据包，非 TCP 流量将继续 OUTPUT 链。
    Chain INPUT (policy ACCEPT 148 packets, 8880 bytes)
     pkts bytes target     prot opt in     out     source               destination
    
    # ISTIO_INBOUND 链：将所有入站流量重定向到 ISTIO_IN_REDIRECT 链上，目的地为 15090，15020，15021端口的流量除外，发送到以上两个端口的流量将返回 iptables 规则链的调用点，即 PREROUTING 链的后继 POSTROUTING。
    Chain ISTIO_INBOUND (1 references)
     pkts bytes target     prot opt in     out     source               destination
        0     0 RETURN     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:15008
        0     0 RETURN     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:22
        0     0 RETURN     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:15090
      143  8580 RETURN     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:15021
        5   300 RETURN     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:15020
        0     0 ISTIO_IN_REDIRECT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0
    
    # ISTIO_IN_REDIRECT 链：将所有入站流量跳转到本地的 15006 端口，至此成功的拦截了流量到sidecar中。
    Chain ISTIO_IN_REDIRECT (3 references)
     pkts bytes target     prot opt in     out     source               destination
        0     0 REDIRECT   tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            redir ports 15006
    ```

12. 被监听在15006端口的envoy进程处理

    ```bash
    $ istioctl pc listener reviews-v3-6c7d64cd96-75f4x.bookinfo --port 15006 -ojson|more
    [
        {
            "name": "virtualInbound",
            "address": {
                "socketAddress": {
                    "address": "0.0.0.0",
                    "portValue": 15006
                }
            },
    ...
    ```

13. VirtualInbound不再转给别的监听器，根据自身过滤器链的匹配条件，请求被Virtual Inbound Listener内部配置的Http connection manager filter处理 ， 该filter设置的路由配置为将其发送给 `inbound|9080|http|reviews.bookinfo.svc.cluster.local`这个inbound的cluster

    ```bash
                {
                    "filterChainMatch": {
                        "destinationPort": 9080
                    },
                    "filters": [
                        {
                            "name": "istio.metadata_exchange",
                            "typedConfig": {
                                "@type": "type.googleapis.com/udpa.type.v1.TypedStruct",
                                "value": {
                                    "protocol": "istio-peer-exchange"
                                }
                            }
                        },
                        {
                            "name": "envoy.filters.network.http_connection_manager",
                            "typedConfig": {
                                "statPrefix": "inbound_0.0.0.0_9080",
                                "routeConfig": {
                                    "name": "inbound|9080|http|reviews.bookinfo.svc.cluster.local",
                                    "virtualHosts": [
                                        {
                                            "name": "inbound|http|9080",
                                            "domains": [
                                                "*"
                                            ],
                                            "routes": [
                                                {
                                                    "name": "default",
                                                    "match": {
                                                        "prefix": "/"
                                                    },
                                                    "route": {
                                                        "cluster": "inbound|9080|http|reviews.bookinfo.svc.cluster.local",
                                                        "timeout": "0s",
                                                        "maxGrpcTimeout": "0s"
                                                    }
    ```

    

14. `inbound|9080|http|reviews.bookinfo.svc.cluster.local`配置的endpoint为本机的`127.0.0.1:9080`,因此转发到Pod内部的Reviews服务的9080端口进行处理。

    ```bash
    $ istioctl pc endpoint reviews-v3-6c7d64cd96-75f4x.bookinfo --cluster "inbound|9080|http|reviews.bookinfo.svc.cluster.local" -ojson
    [
        {
            "name": "inbound|9080|http|reviews.bookinfo.svc.cluster.local",
            "addedViaApi": true,
            "hostStatuses": [
                {
                    "address": {
                        "socketAddress": {
                            "address": "127.0.0.1",
                            "portValue": 9080
                        }
                    },
    ```

    



