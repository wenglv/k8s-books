### 1. 设置hosts解析

操作节点：所有节点（`k8s-master，k8s-node`）均需执行

- **修改hostname**
  hostname必须只能包含小写字母、数字、","、"-"，且开头结尾必须是小写字母或数字

``` python
# 以其中一台master和node为例：

# 在k8s-master1节点
$ hostnamectl set-hostname k8s-master1 #设置master节点的hostname

# 在node-1节点
$ hostnamectl set-hostname k8s-node1 #设置node1节点的hostname

```

- **添加hosts解析**

``` python
$ cat >>/etc/hosts<<EOF
172.21.51.67 k8s-master1
172.21.51.68 k8s-master2
172.21.51.50 k8s-master3
172.21.51.143 k8s-node1
172.21.52.84 k8s-node1
EOF
```

### 2. 调整系统配置

操作节点： 所有的master和node节点（`k8s-master,k8s-node`）需要执行

>本章下述操作均以k8s-master为例，其他节点均是相同的操作（ip和hostname的值换成对应机器的真实值）

- **设置安全组开放端口**

如果节点间无安全组限制（内网机器间可以任意访问），可以忽略，否则，至少保证如下端口可通：
k8s-master节点：TCP：6443，2379，2380，60080，60081UDP协议端口全部打开
k8s-node节点：UDP协议端口全部打开

- **设置iptables**

``` python
iptables -P FORWARD ACCEPT
```

- **关闭swap**

``` python
swapoff -a && sysctl -w vm.swappiness=0
# 防止开机自动挂载 swap 分区
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

- **关闭selinux和防火墙**

``` python
sed -ri 's#(SELINUX=).*#\1disabled#' /etc/selinux/config
setenforce 0
systemctl disable firewalld && systemctl stop firewalld
```

- **修改内核参数**

``` python
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward=1
vm.max_map_count=262144
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
EOF
modprobe br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf
```

- 设置yum源

```bash
$ curl -o /etc/yum.repos.d/Centos-7.repo http://mirrors.aliyun.com/repo/Centos-7.repo
$ curl -o /etc/yum.repos.d/docker-ce.repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
$ yum clean all && yum makecache
$ yum install ipvsadm ipset sysstat conntrack libseccomp -y
```

### 3. 安装docker

操作节点： 所有节点

``` python
 ## 查看所有的可用版本
$ yum list docker-ce --showduplicates | sort -r
##安装旧版本 yum install docker-ce-cli-18.09.9-3.el7  docker-ce-18.09.9-3.el7
## 安装源里最新版本
$ yum install docker-ce-20.10.12 -y

## 配置docker加速
$ mkdir -p /etc/docker
vi /etc/docker/daemon.json
{
  "insecure-registries": [    
    "172.21.51.67:5000" 
  ],                          
  "registry-mirrors" : [
    "https://8xpk5wnt.mirror.aliyuncs.com"
  ],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
## 启动docker
$ systemctl enable docker && systemctl start docker
```

