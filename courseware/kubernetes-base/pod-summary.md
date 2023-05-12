###### 小结

1. 实现k8s平台与特定的容器运行时解耦，提供更加灵活的业务部署方式，引入了Pod概念
2. k8s使用yaml格式定义资源文件，yaml中Map与List的语法，与json做类比
3. 通过kubectl apply| get | exec | logs | delete 等操作k8s资源，必须指定namespace
4. 每启动一个Pod，为了实现网络空间共享，会先创建Infra容器，并把其他容器网络加入该容器
5. 通过livenessProbe和readinessProbe实现Pod的存活性和就绪健康检查
6. 通过requests和limit分别限定容器初始资源申请与最高上限资源申请
7. Pod通过initContainer和lifecycle分别来执行初始化、pod启动和删除时候的操作，使得功能更加全面和灵活
8. 编写yaml讲究方法，学习k8s，养成从官方网站查询知识的习惯

做了哪些工作：

1. 定义Pod.yaml，将myblog和mysql打包在同一个Pod中，使用myblog使用localhost访问mysql
2. mysql数据持久化，为myblog业务应用添加了健康检查和资源限制
3. 将myblog与mysql拆分，使用独立的Pod管理
4. yaml文件中的环境变量存在账号密码明文等敏感信息，使用configMap和Secret来统一配置，优化部署

只使用Pod, 面临的问题:

1. 业务应用启动多个副本
2. Pod重建后IP会变化，外部如何访问Pod服务
3. 运行业务Pod的某个节点挂了，可以自动帮我把Pod转移到集群中的可用节点启动起来
4. 我的业务应用功能是收集节点监控数据,需要把Pod运行在k8集群的各个节点上

