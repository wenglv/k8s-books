#### 小结

1. 核心讲如何通过k8s管理业务应用
2. 介绍k8s的架构、核心组件和工作流程，使用kubeadm快速安装k8s集群
3. 定义Pod.yaml，将myblog和mysql打包在同一个Pod中，myblog使用localhost访问mysql
4. mysql数据持久化，为myblog业务应用添加了健康检查和资源限制
5. 将myblog与mysql拆分，使用独立的Pod管理
6. yaml文件中的环境变量存在账号密码明文等敏感信息，使用configMap和Secret来统一配置，优化部署
7. 只用Pod去直接管理业务应用，对于多副本的需求，很难实现，因此使用Deployment Workload
8. 有了多副本，多个Pod如何去实现LB入口，因此引入了Service的资源类型，有CLusterIp和NodePort
9. ClusterIP是四层的IP地址，不固定，不具备跨环境迁移，因此利用coredns实现集群内服务发现，组件之间直接通过Service名称通信，实现配置的去IP化
10. 对Django应用做改造，django直接使用mysql:3306实现数据库访问
11. 为了实现在集群外部对集群内服务的访问，因此创建NodePort类型的Service
12. 介绍了Service的实现原理，通过kube-proxy利用iptables或者ipvs维护服务访问规则，实现虚拟IP转发到具体Pod的需求
13. 为了实现集群外使用域名访问myblog，因此引入Ingress资源，通过定义访问规则，实现七层代理
14. 考虑真实的场景，对Ingress的使用做了拓展，介绍多path转发及nginx URL重写的实现