操作节点：只在master节点（`k8s-master`）执行，CNI

- 下载flannel的yaml文件

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

- 修改配置，指定网卡名称，大概在文件的190行，添加一行配置：

```bash
$ vi kube-flannel.yml
...      
      containers:
      - name: kube-flannel
        image: rancher/mirrored-flannelcni-flannel:v0.16.1
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=eth0  # 如果机器存在多网卡的话，指定内网网卡的名称，默认不指定的话会找第一块网卡
        resources:
          requests:
            cpu: "100m"
...
```

- 执行安装flannel网络插件

```bash
# 先拉取镜像,此过程国内速度比较慢
$ docker pull rancher/mirrored-flannelcni-flannel:v0.16.1
# 执行flannel安装
$ kubectl apply -f kube-flannel.yml
```

