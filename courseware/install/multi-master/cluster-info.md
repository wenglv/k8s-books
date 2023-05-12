### 1. 节点规划

部署k8s集群的节点按照用途可以划分为如下2类角色：

- **master**：集群的master节点，集群的初始化节点，基础配置不低于2C4G
- **node**：集群的node节点，可以多台，基础配置不低于2C4G

**本例为了演示node节点的添加，会部署一台master+2台node**，节点规划如下：

|   主机名    |    节点ip     |  角色  |                           部署组件                           |
| :---------: | :-----------: | :----: | :----------------------------------------------------------: |
| k8s-master1 | 172.21.51.67  | master | etcd, kube-apiserver, kube-controller-manager, kubectl, kubeadm, kubelet, kube-proxy, flannel |
| k8s-master2 | 172.21.51.68  | master | etcd, kube-apiserver, kube-controller-manager, kubectl, kubeadm, kubelet, kube-proxy, flannel |
| k8s-master3 | 172.21.51.55  | master | etcd, kube-apiserver, kube-controller-manager, kubectl, kubeadm, kubelet, kube-proxy, flannel |
|             | 172.21.51.120 |  VIP   |                  作为3台master节点的LB使用                   |
|  k8s-node1  | 172.21.51.143 |  node  |            kubectl, kubelet, kube-proxy, flannel             |
|  k8s-node2  | 172.21.52.84  |  node  |            kubectl, kubelet, kube-proxy, flannel             |

### 2. 组件版本

|    组件    |               版本                | 说明                                    |
| :--------: | :-------------------------------: | :-------------------------------------- |
|   CentOS   |             7.8.2003              |                                         |
|   Kernel   | Linux 3.10.0-1127.10.1.el7.x86_64 |                                         |
|    etcd    |             3.4.13-0              | 使用Pod方式部署，默认数据挂载到本地路径 |
|  coredns   |               1.8.0               |                                         |
|  kubeadm   |              v1.21.5              |                                         |
|  kubectl   |              v1.21.5              |                                         |
|  kubelet   |              v1.21.5              |                                         |
| kube-proxy |              v1.21.5              |                                         |
|  flannel   |              v0.11.0              |                                         |

