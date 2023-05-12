
### 安装 kubeadm, kubelet 和 kubectl

操作节点： 所有的master和slave节点(`k8s-master,k8s-slave`) 需要执行

``` bash
$ yum install -y kubelet-1.21.5 kubeadm-1.21.5 kubectl-1.21.5 --disableexcludes=kubernetes
## 查看kubeadm 版本
$ kubeadm version
## 设置kubelet开机启动
$ systemctl enable kubelet 
```

### 初始化配置文件

操作节点： 只在master节点（`k8s-master`）执行

``` yaml
$ kubeadm config print init-defaults > kubeadm.yaml
$ cat kubeadm.yaml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.21.51.143
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-master
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: 1.21.5
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

>  对于上面的资源清单的文档比较杂，要想完整了解上面的资源对象对应的属性，可以查看对应的 godoc 文档，地址: https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2。 

### 提前下载镜像

操作节点：只在master节点（`k8s-master`）执行

``` python
  # 查看需要使用的镜像列表,若无问题，将得到如下列表
$ kubeadm config images list --config kubeadm.yaml
registry.aliyuncs.com/google_containers/kube-apiserver:v1.16.0
registry.aliyuncs.com/google_containers/kube-controller-manager:v1.16.0
registry.aliyuncs.com/google_containers/kube-scheduler:v1.16.0
registry.aliyuncs.com/google_containers/kube-proxy:v1.16.0
registry.aliyuncs.com/google_containers/pause:3.1
registry.aliyuncs.com/google_containers/etcd:3.3.15-0
registry.aliyuncs.com/google_containers/coredns:1.21.5
  # 提前下载镜像到本地
$ kubeadm config images pull --config kubeadm.yaml
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-apiserver:v1.16.0
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-controller-manager:v1.16.0
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-scheduler:v1.16.0
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-proxy:v1.16.0
[config/images] Pulled registry.aliyuncs.com/google_containers/pause:3.1
[config/images] Pulled registry.aliyuncs.com/google_containers/etcd:3.3.15-0
[config/images] Pulled registry.aliyuncs.com/google_containers/coredns:1.21.5
```



### 初始化master节点

操作节点：只在master节点（`k8s-master`）执行

``` python
$ kubeadm init --config kubeadm.yaml
```

若初始化成功后，最后会提示如下信息：

``` python
...
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.21.51.143:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:1c4305f032f4bf534f628c32f5039084f4b103c922ff71b12a5f0f98d1ca9a4f
```

接下来按照上述提示信息操作，配置kubectl客户端的认证

``` python
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> **⚠️注意：**此时使用 kubectl get nodes查看节点应该处于notReady状态，因为还未配置网络插件
>
> 若执行初始化过程中出错，根据错误信息调整后，执行kubeadm reset后再次执行init操作即可

### 添加slave节点到集群中

操作节点：所有的slave节点（`k8s-slave`）需要执行
在每台slave节点，执行如下命令，该命令是在kubeadm init成功后提示信息中打印出来的，需要替换成实际init后打印出的命令。

``` python
kubeadm join 172.21.51.143:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:1c4305f032f4bf534f628c32f5039084f4b103c922ff71b12a5f0f98d1ca9a4f
```

如果忘记添加命令，可以通过如下命令生成：

```bash
$ kubeadm token create --print-join-command
```

