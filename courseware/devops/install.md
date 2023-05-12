##### Kubernetes环境中部署jenkins

[其他部署方式](https://jenkins.io/zh/doc/book/installing/)

注意点：

1. 第一次启动很慢
2. 因为后面Jenkins会与kubernetes集群进行集成，会需要调用kubernetes集群的api，因此安装的时候创建了ServiceAccount并赋予了cluster-admin的权限
3. 初始化容器来设置权限
4. ingress来外部访问
5. 数据存储通过pvc挂载到宿主机中

`jenkins/jenkins-all.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkins
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs
  resources:
    requests:
      storage: 200Gi
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins-crb
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: jenkins
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins-master
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      devops: jenkins-master
  template:
    metadata:
      labels:
        devops: jenkins-master
    spec:
      serviceAccount: jenkins #Pod 需要使用的服务账号
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 1000:1000 /var/jenkins_home"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: jenkinshome
          mountPath: /var/jenkins_home
      containers:
      - name: jenkins
        image: jenkinsci/blueocean:1.25.2
        imagePullPolicy: IfNotPresent
        ports:
        - name: http #Jenkins Master Web 服务端口
          containerPort: 8080
        - name: slavelistener #Jenkins Master 供未来 Slave 连接的端口
          containerPort: 50000
        volumeMounts:
        - name: jenkinshome
          mountPath: /var/jenkins_home
        env:
        - name: JAVA_OPTS
          value: "-Xms4096m -Xmx5120m -Duser.timezone=Asia/Shanghai -Dhudson.model.DirectoryBrowserSupport.CSP="
      volumes:
      - name: jenkinshome
        persistentVolumeClaim:
          claimName: jenkins
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: slavelistener
    port: 50000
    targetPort: 50000
  type: ClusterIP
  selector:
    devops: jenkins-master
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jenkins-web
  namespace: jenkins
spec:
  rules:
  - host: jenkins.luffy.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: jenkins
            port:
              number: 8080
```

创建服务：

```bash
## 部署服务
$ kubectl create -f jenkins-all.yaml
## 查看服务
$ kubectl -n jenkins get po
NAME                              READY   STATUS    RESTARTS   AGE
jenkins-master-767df9b574-lgdr5   1/1     Running   0          20s

# 查看日志，第一次启动提示需要完成初始化设置
$ kubectl -n jenkins logs -f jenkins-master-767df9b574-lgdr5
......
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

5396b4e1c395450f8360efd8ee641b18

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************

```

访问服务：

配置hosts解析，`172.21.51.143 jenkins.luffy.com`，然后使用浏览器域名访问服务。第一次访问需要大概几分钟的初始化时间。

![](images\jenkins_setup.jpg)

使用jenkins启动日志中的密码，或者执行下面的命令获取解锁的管理员密码：

```bash
$ kubectl -n jenkins exec jenkins-master-767df9b574-lgdr5 bash 
/ # cat /var/jenkins_home/secrets/initialAdminPassword
35b083de1d25409eaef57255e0da481a

```

点击叉号，跳过选择安装推荐的插件环节，直接进入Jenkins。由于默认的插件地址安装非常慢，我们可以替换成国内清华的源，进入 jenkins 工作目录，目录下面有一个 `updates` 的目录，下面有一个 `default.json` 文件，我们执行下面的命令替换插件地址：

```bash
$ cd /var/jenkins_home/updates
$ sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json 
$ sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json

```

> 暂时先不用重新启动pod，汉化后一起重启。

选择右上角admin->configure->password重新设置管理员密码，设置完后，会退出要求重新登录，使用admin/xxxxxx(新密码)，登录即可。

![](images\jenkins-mainpage.jpg)

##### 安装汉化插件

Jenkins -> manage Jenkins -> Plugin Manager -> Avaliable，搜索 `chinese`关键字

![](images\jenkins-install-plugins.jpg)

选中后，选择[Install without restart]，等待下载完成，然后点击[ Restart Jenkins when installation is complete and no jobs are running ]，让Jenkins自动重启



启动后，界面默认变成中文。

