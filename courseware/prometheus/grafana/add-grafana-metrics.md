###### 导入Dashboard的配置

dashboard： https://grafana.com/grafana/dashboards 

- Node Exporter  https://grafana.com/grafana/dashboards/8919 
- Kubenetes： https://grafana.com/grafana/dashboards/8588 废弃
- Kubenetes： https://grafana.com/grafana/dashboards/13105



###### DevOpsProdigy KubeGraf插件的使用

除了直接导入Dashboard，我们还可以通过安装插件的方式获得，Configuration -> Plugins可以查看已安装的插件，通过 [官方插件列表](https://grafana.com/grafana/plugins?utm_source=grafana_plugin_list) 我们可以获取更多可用插件。

Kubernetes相关的插件：

- [grafana-kubernetes-app](https://grafana.com/grafana/plugins/grafana-kubernetes-app)
- [devopsprodigy-kubegraf-app](https://grafana.com/grafana/plugins/devopsprodigy-kubegraf-app)

[DevOpsProdigy KubeGraf](https://grafana.com/grafana/plugins/devopsprodigy-kubegraf-app) 是一个非常优秀的 Grafana Kubernetes 插件，是 Grafana 官方的 Kubernetes 插件的升级版本，该插件可以用来可视化和分析 Kubernetes 集群的性能，通过各种图形直观的展示了 Kubernetes 集群的主要服务的指标和特征，还可以用于检查应用程序的生命周期和错误日志。



```bash
# 进入grafana容器内部执行安装
$ kubectl -n monitor exec -ti grafana-594f447d6c-jmjsw bash
bash-5.0# grafana-cli plugins install devopsprodigy-kubegraf-app 1.5.2
installing devopsprodigy-kubegraf-app @ 1.5.2
from: https://grafana.com/api/plugins/devopsprodigy-kubegraf-app/versions/1.5.2/download
into: /var/lib/grafana/plugins

✔ Installed devopsprodigy-kubegraf-app successfully

Restart grafana after installing plugins . <service grafana-server restart>

# 也可以下载离线包进行安装

# 重建pod生效
$ kubectl -n monitor delete po grafana-594f447d6c-jmjsw
```



登录grafana界面，Configuration -> Plugins 中找到安装的插件，点击插件进入插件详情页面，点击 [Enable]按钮启用插件，点击 `Set up your first k8s-cluster` 创建一个新的 Kubernetes 集群: 

- Name：luffy-k8s

- URL：https://kubernetes.default:443

- Access：使用默认的Server(default)

- Skip TLS Verify：勾选，跳过证书合法性校验

- Auth：勾选TLS Client Auth以及With CA Cert，勾选后会下面有三块证书内容需要填写，内容均来自`~/.kube/config`文件，需要对文件中的内容做一次base64 解码

  - CA Cert：使用config文件中的`certificate-authority-data`对应的内容
  - Client Cert：使用config文件中的`client-certificate-data`对应的内容
  - Client Key：使用config文件中的`client-key-data`对应的内容

  

> 面板没有数据怎么办？

- DaemonSet

  label_values(kube_pod_info{namespace="$namespace",pod=~"$daemonset-.*"},pod)

- Deployment

  label_values(kube_pod_info{namespace="$namespace",pod=~"$deployment-.*"},pod)

- Pod

  label_values(kube_pod_info{namespace="$namespace"},pod)

###### 自定义监控面板

通用的监控需求基本上都可以使用第三方的Dashboard来解决，对于业务应用自己实现的指标的监控面板，则需要我们手动进行创建。

调试Panel：直接输入Metrics，查询数据。

如，输入`node_load1`来查看集群节点最近1分钟的平均负载，直接保存即可生成一个panel



如何根据字段过滤，实现联动效果？

比如想实现根据集群节点名称进行过滤，可以通过如下方式：

- 设置 -> Variables -> Add Variable，添加一个变量node，

  - Name：node
  - Label：选择节点
  - Data Source：Prometheus
  - Query：label_values(kube_node_info,node)，可以在页面下方的`Preview of values`查看到当前变量的可选值
  - Refresh：`On Dashboard Load`
  - Multi-value：true
  - Include All Options：true

- 修改Metrics，$node和变量名字保持一致，意思为自动读取当前设置的节点的名字

  ```bash
  node_load1{instance=~"$node"}
  ```

  

再添加一个面板，使用如下的表达式：

```bash
100-avg(irate(node_cpu_seconds_total{mode="idle",instance=~"$node"}[5m])) by (instance)*100
```

