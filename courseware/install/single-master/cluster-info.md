
### 节点规划

部署k8s集群的节点按照用途可以划分为如下2类角色：

- **master**：集群的master节点，集群的初始化节点，基础配置不低于2C4G
- **slave**：集群的slave节点，可以多台，基础配置不低于2C4G

**本例为了演示slave节点的添加，会部署一台master+2台slave**，节点规划如下：

|   主机名   |    节点ip     |  角色  |                           部署组件                           |
| :--------: | :-----------: | :----: | :----------------------------------------------------------: |
| k8s-master | 172.21.51.143 | master | etcd, kube-apiserver, kube-controller-manager, kubectl, kubeadm, kubelet, kube-proxy, flannel |
| k8s-slave1 | 172.21.51.67  | slave  |            kubectl, kubelet, kube-proxy, flannel             |
| k8s-slave2 | 172.21.51.68  | slave  |            kubectl, kubelet, kube-proxy, flannel             |

### 组件版本

|    组件    |               版本                | 说明                                    |
| :--------: | :-------------------------------: | :-------------------------------------- |
|   CentOS   |             7.8.2003              |                                         |
|   Kernel   | Linux 3.10.0-1127.10.1.el7.x86_64 |                                         |
|    etcd    |             3.4.13-0              | 使用Pod方式部署，默认数据挂载到本地路径 |
|  coredns   |               1.7.0               |                                         |
|  kubeadm   |              v1.21.5              |                                         |
|  kubectl   |              v1.21.5              |                                         |
|  kubelet   |              v1.21.5              |                                         |
| kube-proxy |              v1.21.5              |                                         |
|  flannel   |              v0.11.0              |                                         |

