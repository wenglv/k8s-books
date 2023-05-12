###### 认识envoy

 `Envoy` 是为云原生应用设计的代理。

可以和nginx做类比： https://fuckcloudnative.io/posts/migrating-from-nginx-to-envoy/

```bash
$ docker run -d --name envoy -v `pwd`/envoy.yaml:/etc/envoy/envoy.yaml -p 10000:10000 envoyproxy/envoy-alpine:v1.15.2

$ curl localhost:10000
```



`envoy.yaml`

```yaml
admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address: { address: 127.0.0.1, port_value: 9901 }

static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 10000 }
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          stat_prefix: ingress_http
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { cluster: some_service }
          http_filters:
          - name: envoy.router
  clusters:
  - name: some_service
    connect_timeout: 2s
    type: STATIC
    lb_policy: ROUND_ROBIN
    hosts: [{ socket_address: { address: 10.103.211.217, port_value: 9999 }}]
```

脑补一下网络代理程序的流程，比如作为一个代理，首先要能获取请求流量，通常是采用监听端口的方式实现；其次拿到请求数据后需要对其做微处理，例如附加 `Header` 或校验某个 `Header` 字段的内容等，这里针对来源数据的层次不同，可以分为 `L3/L4/L7`，然后将请求转发出去；转发这里又可以衍生出如果后端是一个集群，需要从中挑选一台机器，如何挑选又涉及到负载均衡等。 

- `listener` : Envoy 的监听地址。Envoy 会暴露一个或多个 Listener 来监听客户端的请求。
- `filter` : 过滤器。在 Envoy 中指的是一些“可插拔”和可组合的逻辑处理层，是 Envoy 核心逻辑处理单元。
- `route_config` : 路由规则配置。即将请求路由到后端的哪个集群。
- `cluster` : 服务提供方集群。Envoy 通过服务发现定位集群成员并获取服务，具体路由到哪个集群成员由负载均衡策略决定。

###### envoy的xDS

Envoy的启动配置文件分为两种方式：静态配置和动态配置。

- 静态配置是将所有信息都放在配置文件中，启动的时候直接加载。
- 动态配置需要提供一个Envoy的服务端，用于动态生成Envoy需要的服务发现接口，这里叫XDS，通过发现服务来动态的调整配置信息，Istio就是实现了v2的API。

Envoy 接收到请求后，会先走 `FilterChain`，通过各种 L3/L4/L7 Filter 对请求进行微处理，然后再路由到指定的集群，并通过负载均衡获取一个目标地址，最后再转发出去。

其中每一个环节可以静态配置，也可以动态服务发现，也就是所谓的 `xDS`。这里的 `x` 是一个代词，类似云计算里的 `XaaS` 可以指代 IaaS、PaaS、SaaS 等。

![](images\xds.webp)



所以，envoy的架构大致的样子如下：

![](images\arct-envoy.png)

**Downstream**

下游（downstream）主机连接到 Envoy，发送请求并或获得响应。

**Upstream**

上游（upstream）主机获取来自 Envoy 的链接请求和响应。

**监听器**

- 除了过滤器链之外，还有一种过滤器叫**监听器过滤器**（Listener filters），它会在过滤器链之前执行，用于操纵连接的**元数据**。这样做的目的是，无需更改 Envoy 的核心代码就可以方便地集成更多功能。
- 每个监听器都可以配置多个过[滤器链（Filter Chains）](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/listener/v3/listener_components.proto#envoy-v3-api-msg-config-listener-v3-filterchain)，监听器会根据 `filter_chain_match` 中的[匹配条件](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/listener/v3/listener_components.proto#envoy-v3-api-msg-config-listener-v3-filterchainmatch)将流量转交到对应的过滤器链，其中每一个过滤器链都由一个或多个**网络过滤器**（`Network filters`）组成。这些过滤器用于执行不同的代理任务，如速率限制，`TLS` 客户端认证，`HTTP` 连接管理，`MongoDB` 嗅探，原始 TCP 代理等。



###### envoy在微服务治理中的工作环境

可以在服务旁运行，以平台无关的方式提供必要的特性，所有到服务的流量都通过 `Envoy` 代理，这里 `Envoy` 扮演的就是 `Sidecar` 的角色。 

![](images\sidecar.png)

针对于k8s的pod来讲：

![](images\20200503110532.png)

在istio中，envoy的位置：

![](images\istio-mesh-arch.png)



很明显，istio中，envoy进行流量治理，更多的使用的是XDS进行配置更新，而我们知道，XDS需要有服务端来提供接口，istiod中的pilot组件则提供了xDS服务端接口的实现 。

