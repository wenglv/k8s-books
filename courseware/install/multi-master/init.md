### 1. 安装 kubeadm, kubelet 和 kubectl

操作节点： 所有的master和node节点(`k8s-master,k8s-node`) 需要执行

``` bash
$ yum install -y kubelet-1.21.5 kubeadm-1.21.5 kubectl-1.21.5 --disableexcludes=kubernetes
## 查看kubeadm 版本
$ kubeadm version
## 设置kubelet开机启动
$ cat >/etc/sysconfig/kubelet<<EOF
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.2"
EOF
$ systemctl enable kubelet 
```


### 2. 安装配置haproxy、keepalived

操作节点： 所有的master

```bash
$ yum install keepalived haproxy -y

# 所有master节点执行,注意替换最后的master节点IP地址
$ vi /etc/haproxy/haproxy.cfg
global
  maxconn  2000
  ulimit-n  16384
  log  127.0.0.1 local0 err
  stats timeout 30s
defaults
  log global
  mode  http
  option  httplog
  timeout connect 5000
  timeout client  50000
  timeout server  50000
  timeout http-request 15s
timeout http-keep-alive 15s
frontend monitor-in
  bind *:33305
  mode http
  option httplog
  monitor-uri /monitor
frontend k8s-master
  bind 0.0.0.0:7443
  bind 127.0.0.1:7443
  mode tcp
  option tcplog
  tcp-request inspect-delay 5s
  default_backend k8s-master
backend k8s-master
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server k8s-master1    172.21.51.67:6443  check
  server k8s-master2    172.21.51.68:6443   check
  server k8s-master3    172.21.51.55:6443   check 
  
  
# 在k8s-master1节点，注意mcast_src_ip换成实际的master1ip地址，virtual_ipaddress换成lb地址
$ vi /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
script_user root
    enable_script_security
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2
    rise 1
}
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    mcast_src_ip 172.21.51.67
    virtual_router_id 60
    priority 101
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        172.21.51.120
    }
    track_script {
       chk_apiserver
    }
}
# 在k8s-master2和k8s-master3分别创建/etc/keepalived/keepalived.conf，注意修改mcast_src_ip和virtual_ipaddress和state 改为BACKUP
# 在k8s-master2节点
$ cat /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
script_user root
    enable_script_security
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2
rise 1
}
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    mcast_src_ip 172.21.51.68
    virtual_router_id 60
    priority 101
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        172.21.51.120
    }
    track_script {
       chk_apiserver
    }
}



#所有master节点配置KeepAlived健康检查文件：
$ cat /etc/keepalived/check_apiserver.sh
#!/bin/bash
err=0
for k in $(seq 1 3)
do
    check_code=$(pgrep haproxy)
    if [[ $check_code == "" ]]; then
        err=$(expr $err + 1)
        sleep 1
        continue
    else
        err=0
        break
    fi
done
if [[ $err != "0" ]]; then
    echo "systemctl stop keepalived"
    /usr/bin/systemctl stop keepalived
    exit 1
else
    exit 0
fi

# 启动haproxy和keepalived---->所有master节点
$ chmod +x /etc/keepalived/check_apiserver.sh
$ systemctl daemon-reload
$ systemctl enable --now haproxy
$ systemctl enable --now keepalived 


# 测试lbip是否生效
$ telnet 172.21.51.120 7443
```



### 3. 初始化配置文件

操作节点： 只在master节点（`k8s-master`）执行

``` yaml
$ cat kubeadm.yaml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: 7t2weq.bjbawausm0jaxury
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.21.51.67
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-master01
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  certSANs:
  - 172.21.51.120
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: 172.21.51.120:7443
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.21.5
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

>  对于上面的资源清单的文档比较杂，要想完整了解上面的资源对象对应的属性，可以查看对应的 godoc 文档，地址: https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2。 

### 4. 提前下载镜像

操作节点：所有master节点执行

``` python
  # 查看需要使用的镜像列表,若无问题，将得到如下列表
$ kubeadm config images list --config kubeadm.yaml
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.21.5
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.21.5
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.21.5
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.21.5
registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.4.1
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.0
  # 提前下载镜像到本地
$ kubeadm config images pull --config kubeadm.yaml
```



### 5. 初始化master节点

操作节点：只在master节点（`k8s-master`）执行

``` python
$ kubeadm init --config kubeadm.yaml --upload-certs
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

kubeadm join 172.21.51.120:7443 --token 7t2weq.bjbawausm0jaxury         --discovery-token-ca-cert-hash sha256:b0d875f1dafe9f479b23603c3424cad5e0e3aa0a47a8274f9d24432e97e3dbde         --control-plane --certificate-key 0ea981458813160b6fbc572d415e14cbc28c4bf958a765a7bc989b7ecc5dcdd6
```

接下来按照上述提示信息操作，配置kubectl客户端的认证

``` python
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> 若执行初始化过程中出错，根据错误信息调整后，执行 kubeadm reset -f ; ipvsadm --clear  ; rm -rf ~/.kube

### 6. 添加其他master节点到集群中

```bash
$ kubeadm join 172.21.51.120:7443 --token 7t2weq.bjbawausm0jaxury         --discovery-token-ca-cert-hash sha256:b0d875f1dafe9f479b23603c3424cad5e0e3aa0a47a8274f9d24432e97e3dbde         --control-plane --certificate-key 0ea981458813160b6fbc572d415e14cbc28c4bf958a765a7bc989b7ecc5dcdd6
```



### 7. 添加node节点到集群中

操作节点：所有的node节点（`k8s-node`）需要执行
在每台node节点，执行如下命令，该命令是在kubeadm init成功后提示信息中打印出来的，需要替换成实际init后打印出的命令。

``` python
kubeadm join 172.21.51.120:7443 --token 7t2weq.bjbawausm0jaxury         --discovery-token-ca-cert-hash sha256:b0d875f1dafe9f479b23603c3424cad5e0e3aa0a47a8274f9d24432e97e3dbde      
```

如果忘记添加命令，可以通过如下命令生成：

```bash
$ kubeadm token create --print-join-command
```

