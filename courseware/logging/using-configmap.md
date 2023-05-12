##### ConfigMap的配置文件挂载使用场景

开始之前，我们先来回顾一下，configmap的常用的挂载场景。

###### 场景一：单文件挂载到空目录

假如业务应用有一个配置文件，名为 `application.yml`，如果想将此配置挂载到pod的`/etc/application/`目录中。

`application.yml`的内容为：

```bash
$ cat application.yml
spring:
  application:
    name: svca-service
  cloud:
    config:
      uri: http://config:8888
      fail-fast: true
      username: user
      password: ${CONFIG_SERVER_PASSWORD:password}
      retry:
        initial-interval: 2000
        max-interval: 10000
        multiplier: 2
        max-attempts: 10

```

该配置文件在k8s中可以通过configmap来管理，通常我们有如下两种方式来管理配置文件：

- 通过kubectl命令行来生成configmap

  ```bash
  # 通过文件直接创建
  $ kubectl -n default create configmap application-config --from-file=application.yml
  
  # 会生成配置文件，查看内容，configmap的key为文件名字
  $ kubectl -n default get cm application-config -oyaml
  ```

  

- 通过yaml文件直接创建

  ```bash
  $ cat application-config.yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: application-config
    namespace: default
  data:
    application.yml: |
      spring:
        application:
          name: svca-service
        cloud:
          config:
            uri: http://config:8888
            fail-fast: true
            username: user
            password: ${CONFIG_SERVER_PASSWORD:password}
            retry:
              initial-interval: 2000
              max-interval: 10000
              multiplier: 2
              max-attempts: 10
  
  # 创建configmap
  $ kubectl apply -f application-config.yaml
  ```

准备一个`demo-deployment.yaml`文件，挂载上述configmap到`/etc/application/`中

```bash
$ cat demo-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
  namespace: default
spec:
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      volumes:
      - configMap:
          name: application-config
        name: config
      containers:
      - name: nginx
        image: nginx:alpine
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - mountPath: "/etc/application"
          name: config
```



创建并查看：

```bash
$ kubectl apply -f demo-deployment.yaml
```

修改configmap文件的内容，观察pod中是否自动感知变化：

```bash
$ kubectl edit cm application-config
```



> 整个configmap文件直接挂载到pod中，若configmap变化，pod会自动感知并拉取到pod内部。
>
> 但是pod内的进程不会自动重启，所以很多服务会实现一个内部的reload接口，用来加载最新的配置文件到进程中。



###### 场景二：多文件挂载

假如有多个配置文件，都需要挂载到pod内部，且都在一个目录中

```bash
$ cat application.yml
spring:
  application:
    name: svca-service
  cloud:
    config:
      uri: http://config:8888
      fail-fast: true
      username: user
      password: ${CONFIG_SERVER_PASSWORD:password}
      retry:
        initial-interval: 2000
        max-interval: 10000
        multiplier: 2
        max-attempts: 10

$ cat supervisord.conf
[unix_http_server]
file=/var/run/supervisor.sock
chmod=0700

[supervisord]
logfile=/var/logs/supervisor/supervisord.log
logfile_maxbytes=200MB
logfile_backups=10
loglevel=info
pidfile=/var/run/supervisord.pid
childlogdir=/var/cluster_conf_agent/logs/supervisor
nodaemon=false
```

同样可以使用两种方式创建：

```bash
$ kubectl delete cm application-config

$ kubectl create cm application-config --from-file=application.yml --from-file=supervisord.conf

$ kubectl get cm application-config -oyaml
```

观察Pod已经自动获取到最新的变化

```bash
$ kubectl exec demo-55c649865b-gpkgk ls /etc/application/
application.yml
supervisord.conf
```



此时，是挂载到pod内的空目录中`/etc/application`，假如想挂载到pod已存在的目录中，比如：

```bash
$  kubectl exec   demo-55c649865b-gpkgk ls /etc/profile.d
color_prompt
locale
```

 更改deployment的挂载目录：

```bash
$ cat demo-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
  namespace: default
spec:
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      volumes:
      - configMap:
          name: application-config
        name: config
      containers:
      - name: nginx
        image: nginx:alpine
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - mountPath: "/etc/profile.d"
          name: config
```

重建pod

```bash
$ kubectl apply -f demo-deployment.yaml

# 查看pod内的/etc/profile.d目录，发现已有文件被覆盖
$ kubectl exec demo-77d685b9f7-68qz7 ls /etc/profile.d
application.yml
supervisord.conf
```



###### 场景三  挂载子路径

实现多个配置文件，可以挂载到pod内的不同的目录中。比如：

- `application.yml`挂载到`/etc/application/`
- `supervisord.conf`挂载到`/etc/profile.d`

configmap保持不变，修改deployment文件：

```bash
$ cat demo-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
  namespace: default
spec:
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      volumes:
      - name: config
        configMap:
          name: application-config
          items:
          - key: application.yml
            path: application
          - key: supervisord.conf
            path: supervisord
      containers:
      - name: nginx
        image: nginx:alpine
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - mountPath: "/etc/application/application.yml"
          name: config
          subPath: application
        - mountPath: "/etc/profile.d/supervisord.conf"
          name: config
          subPath: supervisord

```

 测试挂载：

```bash
$ kubectl apply -f demo-deployment.yaml

$ kubectl exec demo-78489c754-shjhz ls /etc/application
application.yml

$ kubectl exec demo-78489c754-shjhz ls /etc/profile.d/
supervisord.conf
color_prompt
locale

```



> 使用subPath挂载到Pod内部的文件，不会自动感知原有ConfigMap的变更

