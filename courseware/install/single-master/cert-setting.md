
使用kubeadm安装的集群，证书默认有效期为1年，可以通过如下方式修改为10年。

```bash
$ cd /etc/kubernetes/pki

# 查看当前证书有效期
$ for i in $(ls *.crt); do echo "===== $i ====="; openssl x509 -in $i -text -noout | grep -A 3 'Validity' ; done

$ mkdir backup_key; cp -rp ./* backup_key/
$ git clone https://github.com/yuyicai/update-kube-cert.git
$ cd update-kube-cert/ 
$ bash update-kubeadm-cert.sh all

# 重建管理服务
$ kubectl -n kube-system delete po kube-apiserver-k8s-master kube-controller-manager-k8s-master kube-scheduler-k8s-master
```

