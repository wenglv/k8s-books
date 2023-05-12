###### 调试envoy

我们知道，envoy的配置非常复杂，直接在config_dump里去跟踪xDS的过程非常繁琐。因此istio提供了调试命令，方便查看envoy的流量处理流程。

```bash
$ istioctl proxy-config -h
```



比如，通过如下命令可以查看envoy的监听器：

```bash
# 查看15001的监听
$ istioctl proxy-config listener front-tomcat-v1-78cf497978-vv9wj.istio-demo --port 15001 -ojson
# virtualOutbound的监听不做请求处理，useOriginalDst: true, 直接转到原始的请求对应的监听器中

# 查看访问端口是9999的监听器
$ istioctl proxy-config listener front-tomcat-v1-78cf497978-ppwwk.istio-demo --port 9999 -ojson
...
    {
        "name": "0.0.0.0_9999",
        "address": {
            "socketAddress": {
                "address": "0.0.0.0",
                "portValue": 9999
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
                            "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
                            "statPrefix": "outbound_0.0.0.0_9999",
                            "rds": {
                                "configSource": {
                                    "ads": {},
                                    "resourceApiVersion": "V3"
                                },
                                "routeConfigName": "9999"
                            },
...
```

> envoy收到请求后，会转给监听器进行处理请求，监听器先匹配address和port和socket都一致的Listener，如果没找到再找port一致，address==0.0.0.0的Listener

发现istio会为网格内的Service Port创建名为`0.0.0.0_<Port>`的虚拟监听器，本例中为`0.0.0.0_9999`。

envoy的15001端口收到请求后，直接转到了`0.0.0.0_9999`，进而转到了`"routeConfigName": "9999"`，即9999这个route中。

下面，看下route的内容：

```bash
$ istioctl pc route front-tomcat-v1-78cf497978-ppwwk.istio-demo --name 9999
NOTE: This output only contains routes loaded via RDS.
NAME     DOMAINS          MATCH     VIRTUAL SERVICE
9999     bill-service     /*        vs-bill-service.istio-demo

# 发现了前面创建的virtual service
$ istioctl pc route front-tomcat-v1-78cf497978-ppwwk.istio-demo --name 9999 -ojson
[
    {
        "name": "9999",
        "virtualHosts": [
            {
                "name": "allow_any",
                "domains": [
                    "*"
                ],
                "routes": [
                    {
                        "name": "allow_any",
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "cluster": "PassthroughCluster",
                            "timeout": "0s",
                            "maxGrpcTimeout": "0s"
                        }
                    }
                ],
                "includeRequestAttemptCount": true
            },
            {
                "name": "bill-service.istio-demo.svc.cluster.local:9999",
                "domains": [
                    "bill-service.istio-demo.svc.cluster.local",
                    "bill-service.istio-demo.svc.cluster.local:9999",
                    "bill-service",
                    "bill-service:9999",
                    "bill-service.istio-demo.svc.cluster",
                    "bill-service.istio-demo.svc.cluster:9999",
                    "bill-service.istio-demo.svc",
                    "bill-service.istio-demo.svc:9999",
                    "bill-service.istio-demo",
                    "bill-service.istio-demo:9999",
                    "10.111.219.247",
                    "10.111.219.247:9999"
                ],
                "routes": [
                    {
                        "name": "bill-service-route",
                        "match": {
                            "prefix": "/"
                        },
                        "route": {
                            "weightedClusters": {
                                "clusters": [
                                    {
                                        "name": "outbound|9999|v1|bill-service.istio-demo.svc.cluster.local",
                                        "weight": 90
                                    },
                                    {
                                        "name": "outbound|9999|v2|bill-service.istio-demo.svc.cluster.local",
                                        "weight": 10
                                    }
                                ]
                            },
...
```

满足访问domains列表的会优先匹配到，我们访问的是`10.111.219.247:9999`，因此匹配`bill-service.istio-demo.svc.cluster.local:9999`这组虚拟hosts，进而使用到基于weight的集群配置。

![](images\9-1.jpg)



我们看到，流量按照预期的配置进行了转发：

```bash
90% -> outbound|9999|v1|bill-service.istio-demo.svc.cluster.local
10% -> outbound|9999|v2|bill-service.istio-demo.svc.cluster.local
```

下面，看一下cluster的具体内容：

```bash
$ istioctl pc cluster front-tomcat-v1-78cf497978-ppwwk.istio-demo --fqdn bill-service.istio-demo.svc.cluster.local -ojson
...
        "name": "outbound|9999|v1|bill-service.istio-demo.svc.cluster.local",
        "type": "EDS",
        "edsClusterConfig": {
            "edsConfig": {
                "ads": {},
                "resourceApiVersion": "V3"
            },
            "serviceName": "outbound|9999|v1|bill-service.istio-demo.svc.cluster.local"
        },
...
```

我们发现，endpoint列表是通过eds获取的，因此，查看endpoint信息：

```bash
$ istioctl pc endpoint front-tomcat-v1-78cf497978-ppwwk.istio-demo  --cluster 'outbound|9999|v1|bill-service.istio-demo.svc.cluster.local' -ojson
[
    {
        "name": "outbound|9999|v1|bill-service.istio-demo.svc.cluster.local",
        "addedViaApi": true,
        "hostStatuses": [
            {
                "address": {
                    "socketAddress": {
                        "address": "10.244.0.17",
                        "portValue": 80
                    }
                },
...
```



目前为止，经过envoy的规则，流量从front-tomcat的pod中知道要发往`10.244.0.7:80` 这个pod地址。前面提到过，envoy不止接管出站流量，入站流量同样会接管。

![](images\20200503110532.png)



下面看下流量到达bill-service-v1的pod后的处理：

先回顾前面的iptables规则，除特殊情况以外，所有的出站流量被监听在15001端口的envoy进程拦截处理，同样的，分析bill-service-v1的iptables规则可以发现，监听在15006端口的envoy进程通过在PREROUTING链上添加规则，同样将进入pod的入站流量做了拦截。

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



15006端口是一个名为 `virtualInbound`虚拟入站监听器，

```bash
$ istioctl pc l bill-service-v1-6c95ccb747-vwt2d.istio-demo --port 15006 -ojson
```

```json
"name": "envoy.filters.network.http_connection_manager",
"typedConfig": {
    "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
    "statPrefix": "inbound_0.0.0.0_80",
    "routeConfig": {
        "name": "inbound|80||",
        "virtualHosts": [
            {
                "name": "inbound|http|9999",
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
                            "cluster": "inbound|80||",
                            "timeout": "0s",
                            "maxStreamDuration": {
                                "maxStreamDuration": "0s",
                                "grpcTimeoutHeaderMax": "0s"
                            }
                        },
                        "decorator": {
                            "operation": "bill-service.istio-demo.svc.cluster.local:9999/*"
                        }
                    }
                ]
            }
        ],
        "validateClusters": false
```



相比于`VirtualOutbound`， `virtualInbound` 不会再次转给别的虚拟监听器，而是直接由本监听器的`filterChains`处理，本例中我们可以发现本机目标地址为80的http请求，转发到了` inbound|9999|http|bill-service.istio-demo.svc.cluster.local `这个集群中。

查看该集群的信息：

```bash
$ istioctl pc cluster bill-service-v1-6c95ccb747-vwt2d.istio-demo -h
$ istioctl pc cluster bill-service-v1-6c95ccb747-vwt2d.istio-demo  --fqdn "inbound|80||" -ojson
[
    {
        "name": "inbound|80||",
        "type": "ORIGINAL_DST",
        "connectTimeout": "10s",
        "lbPolicy": "CLUSTER_PROVIDED",
        "circuitBreakers": {
            "thresholds": [
                {
                    "maxConnections": 4294967295,
                    "maxPendingRequests": 4294967295,
                    "maxRequests": 4294967295,
                    "maxRetries": 4294967295,
                    "trackRemaining": true
                }
            ]
        },
        "cleanupInterval": "60s",
        "upstreamBindConfig": {
            "sourceAddress": {
                "address": "127.0.0.6",
                "portValue": 0
            }
        },
        "metadata": {
            "filterMetadata": {
                "istio": {
                    "services": [
                        {
                            "host": "bill-service.istio-demo.svc.cluster.local",
                            "name": "bill-service",
                            "namespace": "istio-demo"
                        }
                    ]
                }
            }
        }
    }
]
```

