#### 安装

###### 配置宿主机网卡转发

```bash
## 若未配置，需要执行如下
$ cat <<EOF >  /etc/sysctl.d/docker.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward=1
EOF
$ sysctl -p /etc/sysctl.d/docker.conf
```

###### Yum安装配置docker

```bash
## 下载阿里源repo文件
$ curl -o /etc/yum.repos.d/Centos-7.repo http://mirrors.aliyun.com/repo/Centos-7.repo
$ curl -o /etc/yum.repos.d/docker-ce.repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

$ yum clean all && yum makecache
## yum安装
$ yum install docker-ce-20.10.12 -y
## 查看源中可用版本
$ yum list docker-ce --showduplicates | sort -r
## 安装旧版本
##yum install -y docker-ce-18.09.9

## 配置源加速
## https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors
mkdir -p /etc/docker
vi /etc/docker/daemon.json
{
  "registry-mirrors" : [
    "https://8xpk5wnt.mirror.aliyuncs.com"
  ]
}

## 设置开机自启
systemctl enable docker  
systemctl daemon-reload

## 启动docker
systemctl start docker 

## 查看docker信息
docker info

## docker-client
which docker
## docker daemon
ps aux |grep docker
## containerd
ps aux|grep containerd
systemctl status containerd
```
