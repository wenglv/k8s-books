

###### 工作原理

目前为止，我们可以知道大致的工作流程：

- 用户端，通过创建服务治理的规则（VirtualService、DestinationRule等资源类型），存储到ETCD中
- istio控制平面中的Pilot服务监听上述规则，转换成envoy可读的规则配置，通过xDS接口同步给各envoy
- envoy通过xDS获取最新的配置后，动态reload，进而改变流量转发的策略

思考两个问题：

- istio中envoy的动态配置到底长什么样子？
- 在istio的网格内，front-tomcat访问到bill-service，流量的流向是怎么样的？



针对问题1：

每个envoy进程启动的时候，会在`127.0.0.1`启动监听15000端口

```bash
$ kubectl -n istio-demo exec -ti front-tomcat-v1-78cf497978-ppwwk -c istio-proxy bash
# netstat -nltp
# curl localhost:15000/help
# curl localhost:15000/config_dump
```



针对问题2：

```bash
$ kubectl -n istio-demo exec -ti front-tomcat-v1-78cf497978-ppwwk -c front-tomcat bash
# curl bill-service:9999
```

按照之前的认知，

![](images\cluster-ip-route.jpg)



现在为什么流量分配由5：5 变成了9：1？流量经过envoy了的处理

![](images\9-1.jpg)



envoy如何接管由front-tomcat容器发出的请求流量？（istio-init

回顾iptables：

![](images\iptables.jpg)

Istio 给应用 Pod 注入的配置主要包括：

- Init 容器 `istio-init`

  Istio 在 pod 中注入的 Init 容器名为 `istio-init`，作用是为 pod 设置 iptables 端口转发。

  我们在上面 Istio 注入完成后的 YAML 文件中看到了该容器的启动命令是：

  ```bash
  istio-iptables -p 15001 -z 15006 -u 1337 -m REDIRECT -i '*' -x "" -b '*' -d 15090,15021,15020
  ```

  Init 容器的启动入口是 `istio-iptables` 命令行，该命令行工具的用法如下：

  ```bash
  $ istio-iptables [flags]
    -p: 指定重定向所有 TCP 出站流量的 sidecar 端口（默认为 $ENVOY_PORT = 15001）
    -m: 指定入站连接重定向到 sidecar 的模式，“REDIRECT” 或 “TPROXY”（默认为 $ISTIO_INBOUND_INTERCEPTION_MODE)
    -b: 逗号分隔的入站端口列表，其流量将重定向到 Envoy（可选）。使用通配符 “*” 表示重定向所有端口。为空时表示禁用所有入站重定向（默认为 $ISTIO_INBOUND_PORTS）
    -d: 指定要从重定向到 sidecar 中排除的入站端口列表（可选），以逗号格式分隔。使用通配符“*” 表示重定向所有入站流量（默认为 $ISTIO_LOCAL_EXCLUDE_PORTS）
    -o：逗号分隔的出站端口列表，不包括重定向到 Envoy 的端口。
    -i: 指定重定向到 sidecar 的 IP 地址范围（可选），以逗号分隔的 CIDR 格式列表。使用通配符 “*” 表示重定向所有出站流量。空列表将禁用所有出站重定向（默认为 $ISTIO_SERVICE_CIDR）
    -x: 指定将从重定向中排除的 IP 地址范围，以逗号分隔的 CIDR 格式列表。使用通配符 “*” 表示重定向所有出站流量（默认为 $ISTIO_SERVICE_EXCLUDE_CIDR）。
    -k：逗号分隔的虚拟接口列表，其入站流量（来自虚拟机的）将被视为出站流量。
    -g：指定不应用重定向的用户的 GID。(默认值与 -u param 相同)
    -u：指定不应用重定向的用户的 UID。通常情况下，这是代理容器的 UID（默认值是 1337，即 istio-proxy 的 UID）。
    -z: 所有进入 pod/VM 的 TCP 流量应被重定向到的端口（默认 $INBOUND_CAPTURE_PORT = 15006）。
  ```

  以上传入的参数都会重新组装成 [`iptables` ](https://wangchujiang.com/linux-command/c/iptables.html)规则，关于 Istio 中端口用途请参考 [Istio 官方文档](https://istio.io/latest/docs/ops/deployment/requirements/)。 

  这条启动命令的作用是：

  - 将应用容器的所有入站流量都转发到 sidecar的 15006 端口（15090 端口（Envoy Prometheus telemetry）和 15020 端口（Ingress Gateway）除外，15021（sidecar健康检查）端口）
  - 将所有出站流量都重定向到 sidecar 代理（通过 15001 端口）
  - 上述规则对id为1337用户除外，因为1337是istio-proxy自身的流量

  该容器存在的意义就是让 sidecar 代理可以拦截pod所有的入站（inbound）流量以及出站（outbound）流量，这样就可以实现由sidecar容器来接管流量，进而实现流量管控。

  因为 Init 容器初始化完毕后就会自动终止，因为我们无法登陆到容器中查看 iptables 信息，但是 Init 容器初始化结果会保留到应用容器和 sidecar 容器中。

  ```bash
  # 查看front-tomcat服务的istio-proxy容器的id
  $ docker ps |grep front-tomcat
  d02fa8217f2f        consol/tomcat-7.0                                   "/bin/sh -c /opt/tom…"   2 days ago          Up 2 days
                                                k8s_front-tomcat_front-tomcat-v1-78cf497978-ppwwk_istio-demo_f03358b1-ed17-4811-ac7e-9f70e6bd797b_0
  
  # 根据容器id获取front-tomcat容器在宿主机中的进程
  $ docker inspect d02fa8217f2f|grep -i pid
              "Pid": 28834,
              "PidMode": "",
              "PidsLimit": null,
  # 进入该进程的网络命名空间
  $ nsenter -n --target 28834
  # 查看命名空间的iptables规则
  $ iptables -t nat -vnL
  # PREROUTING 链：用于目标地址转换（DNAT），将所有入站 TCP 流量跳转到 ISTIO_INBOUND 链上。
  Chain PREROUTING (policy ACCEPT 148 packets, 8880 bytes)
   pkts bytes target     prot opt in     out     source               destination
    148  8880 ISTIO_INBOUND  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0
  
  # INPUT 链：处理输入数据包，非 TCP 流量将继续 OUTPUT 链。
  Chain INPUT (policy ACCEPT 148 packets, 8880 bytes)
   pkts bytes target     prot opt in     out     source               destination
  
  # OUTPUT 链：将所有出站数据包跳转到 ISTIO_OUTPUT 链上。
  Chain OUTPUT (policy ACCEPT 46 packets, 3926 bytes)
   pkts bytes target     prot opt in     out     source               destination
      8   480 ISTIO_OUTPUT  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0
  # POSTROUTING 链：所有数据包流出网卡时都要先进入POSTROUTING 链，内核根据数据包目的地判断是否需要转发出去，我们看到此处未做任何处理。
  Chain POSTROUTING (policy ACCEPT 46 packets, 3926 bytes)
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




```bash
$ kubectl -n istio-demo exec -ti front-tomcat-v1-78cf497978-ppwwk -c istio-proxy bash
istio-proxy@front-tomcat-v1-78cf497978-ppwwk:/$ netstat -nltp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:15000         0.0.0.0:*               LISTEN      16/envoy
tcp        0      0 0.0.0.0:15001           0.0.0.0:*               LISTEN      16/envoy
tcp        0      0 0.0.0.0:15006           0.0.0.0:*               LISTEN      16/envoy
tcp        0      0 127.0.0.1:8005          0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:8009            0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:8778            0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:15021           0.0.0.0:*               LISTEN      16/envoy
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:15090           0.0.0.0:*               LISTEN      16/envoy
tcp6       0      0 :::15020                :::*                    LISTEN      1/pilot-agent
```

说明pod内的出站流量请求被监听在15001端口的envoy的进程接收到，进而就走到了envoy的Listener -> route -> cluster -> endpoint 转发流程。

问题就转变为：如何查看envoy的配置，跟踪转发的过程？

