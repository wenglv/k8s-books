##### Jenkins基本使用演示

###### 演示目标

- 代码提交gitlab，自动触发Jenkins任务
- Jenkins任务完成后发送钉钉消息通知

###### 演示准备

*gitlab代码仓库搭建*

 https://github.com/sameersbn/docker-gitlab 

```bash
## 全量部署的组件
$ gitlab-ctl status
run: alertmanager: (pid 1987) 27s; run: log: (pid 1986) 27s
run: gitaly: (pid 1950) 28s; run: log: (pid 1949) 28s
run: gitlab-exporter: (pid 1985) 27s; run: log: (pid 1984) 27s
run: gitlab-workhorse: (pid 1956) 28s; run: log: (pid 1955) 28s
run: logrotate: (pid 1960) 28s; run: log: (pid 1959) 28s
run: nginx: (pid 2439) 1s; run: log: (pid 1990) 27s
run: node-exporter: (pid 1963) 28s; run: log: (pid 1962) 28s
run: postgres-exporter: (pid 1989) 27s; run: log: (pid 1988) 27s
run: postgresql: (pid 1945) 28s; run: log: (pid 1944) 28s
run: prometheus: (pid 1973) 28s; run: log: (pid 1972) 28s
run: puma: (pid 1968) 28s; run: log: (pid 1966) 28s
run: redis: (pid 1952) 28s; run: log: (pid 1951) 28s
run: redis-exporter: (pid 1971) 28s; run: log: (pid 1964) 28s
run: sidekiq: (pid 1969) 28s; run: log: (pid 1967) 28s
```

部署分析：

1. 依赖postgres
2. 依赖redis

使用k8s部署：

1. 准备secret文件

   ```bash
   $ cat gitlab-secret.txt
   postgres.user.root=root
   postgres.pwd.root=1qaz2wsx
   
   $ kubectl -n jenkins create secret generic gitlab-secret --from-env-file=gitlab-secret.txt
   ```

   

2. 部署postgres

   注意点：

   - 使用secret来引用账户密码

```bash
$ cat postgres.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
  namespace: jenkins
spec:
  ports:
  - name: server
    port: 5432
    targetPort: 5432
    protocol: TCP
  selector:
    app: postgres
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgredb
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs
  resources:
    requests:
      storage: 200Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: jenkins
  name: postgres
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      tolerations:
      - operator: "Exists"
      containers:
      - name: postgres
        image: postgres:11.4
        imagePullPolicy: "IfNotPresent"
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER           #PostgreSQL 用户名
          valueFrom:
            secretKeyRef:
              name: gitlab-secret
              key: postgres.user.root
        - name: POSTGRES_PASSWORD       #PostgreSQL 密码
          valueFrom:
            secretKeyRef:
              name: gitlab-secret
              key: postgres.pwd.root
        resources:
          limits:
            cpu: 1000m
            memory: 2048Mi
          requests:
            cpu: 50m
            memory: 100Mi
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgredb
      volumes:
      - name: postgredb
        persistentVolumeClaim:
          claimName: postgredb
   
   
   #创建postgres
   $ kubectl create -f postgres.yaml
   
   # 创建数据库gitlab,为后面部署gitlab组件使用
   $ kubectl -n jenkins exec -ti postgres-7ff9b49f4c-nt8zh bash
   root@postgres-7ff9b49f4c-nt8zh:/# psql
   root=# create database gitlab;
   CREATE DATABASE
```



3. 部署redis

   ```bash
   $ cat redis.yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: redis
     labels:
       app: redis
     namespace: jenkins
   spec:
     ports:
     - name: server
       port: 6379
       targetPort: 6379
       protocol: TCP
     selector:
       app: redis
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     namespace: jenkins
     name: redis
     labels:
       app: redis
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: redis
     template:
       metadata:
         labels:
           app: redis
       spec:
         tolerations:
         - operator: "Exists"
         containers:
         - name: redis
           image: sameersbn/redis:4.0.9-2
           imagePullPolicy: "IfNotPresent"
           ports:
           - containerPort: 6379
           resources:
             limits:
               cpu: 1000m
               memory: 2048Mi
             requests:
               cpu: 50m
               memory: 100Mi
               
   # 创建
   $ kubectl create -f redis.yaml
   ```

   

4. 部署gitlab

   注意点：

   - 使用ingress暴漏服务
   - 添加annotation，指定nginx端上传大小限制，否则推送代码时会默认被限制1m大小，相当于给nginx设置client_max_body_size的限制大小
   - 使用服务发现地址来访问postgres和redis
   - 在secret中引用数据库账户和密码
   - 数据库名称为gitlab

```bash
$ cat gitlab.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitlab
  namespace: jenkins
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  rules:
  - host: gitlab.luffy.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gitlab
            port:
              number: 80
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: gitlab
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
kind: Service
metadata:
  name: gitlab
  labels:
    app: gitlab
  namespace: jenkins
spec:
  ports:
  - name: server
    port: 80
    targetPort: 80
    protocol: TCP
  selector:
    app: gitlab
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: jenkins
  name: gitlab
  labels:
    app: gitlab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitlab
  template:
    metadata:
      labels:
        app: gitlab
    spec:
      tolerations:
      - operator: "Exists"
      containers:
      - name: gitlab
        image:  sameersbn/gitlab:13.2.2
        imagePullPolicy: "IfNotPresent"
        env:
        - name: GITLAB_HOST
          value: "gitlab.luffy.com"
        - name: GITLAB_PORT
          value: "80"
        - name: GITLAB_SECRETS_DB_KEY_BASE
          value: "long-and-random-alpha-numeric-string"
        - name: GITLAB_SECRETS_DB_KEY_BASE
          value: "long-and-random-alpha-numeric-string"
        - name: GITLAB_SECRETS_SECRET_KEY_BASE
          value: "long-and-random-alpha-numeric-string"
        - name: GITLAB_SECRETS_OTP_KEY_BASE
          value: "long-and-random-alpha-numeric-string"
        - name: DB_HOST
          value: "postgres"
        - name: DB_NAME
          value: "gitlab"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: gitlab-secret
              key: postgres.user.root
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: gitlab-secret
              key: postgres.pwd.root
        - name: REDIS_HOST
          value: "redis"
        - name: REDIS_PORT
          value: "6379"
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 2000m
            memory: 5048Mi
          requests:
            cpu: 100m
            memory: 500Mi
        volumeMounts:
        - mountPath: /home/git/data
          name: data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: gitlab
   # 创建
   $ kubectl create -f gitlab.yaml
```

   

配置hosts解析：

```bash
172.21.51.143 gitlab.luffy.com
```



*设置root密码*

访问http://gitlab.luffy.com，设置管理员密码

*配置k8s-master节点的hosts*

```bash
$ echo "172.21.51.143 gitlab.luffy.com" >>/etc/hosts
```



*myblog项目推送到gitlab*

```bash
mkdir demo
cp -r python-demo demo/
cd demo/myblog
git remote rename origin old-origin
git remote add origin http://gitlab.luffy.com/root/myblog.git
git push -u origin --all
git push -u origin --tags

```

*钉钉推送*

[官方文档](https://ding-doc.dingtalk.com/doc#/serverapi2/qf2nxq)

- 配置机器人

- 试验发送消息

  ```bash
  $ curl 'https://oapi.dingtalk.com/robot/send?access_token=4778abd23dbdbaf66fc6f413e6ab9c0103a039b0054201344a22a5692cdcc54e' \
     -H 'Content-Type: application/json' \
     -d '{"msgtype": "text", 
          "text": {
               "content": "我就是我, 是不一样的烟火"
          }
        }'
  
  ```

###### 演示过程

流程示意图：

![](images\jenkins-gitlab.png)

1. 安装gitlab plugin

   插件中心搜索并安装gitlab，直接安装即可

2. 配置Gitlab

   系统管理->系统配置->Gitlab，其中的API Token，需要从下个步骤中获取

   ![](images\gitlab-connection.jpg)

3. 获取AccessToken

   登录gitlab，选择user->Settings->access tokens新建一个访问token

4. 配置host解析

   由于我们的Jenkins和gitlab域名是本地解析，因此需要让gitlab和Jenkins服务可以解析到对方的域名。两种方式：

   - 在容器内配置hosts

   - 配置coredns的静态解析

     ```bash
             hosts {
                 172.21.51.143 jenkins.luffy.com  gitlab.luffy.com
                 fallthrough
             }
     ```

5. 创建自由风格项目

   - gitlab connection 选择为刚创建的gitlab
   - 源码管理选择Git，填项项目地址
   - 新建一个 Credentials 认证，使用用户名密码方式，配置gitlab的用户和密码
   - 构建触发器选择 Build when a change is pushed to GitLab 
   - 生成一个Secret token
   - 保存

6. 到gitlab配置webhook

   - 进入项目下settings->Integrations
   - URL： http://jenkins.luffy.com/project/free 
   - Secret Token 填入在Jenkins端生成的token
   - Add webhook
   - test push events，报错：Requests to the local network are not allowed 

7. 设置gitlab允许向本地网络发送webhook请求

   访问 Admin Aera -> Settings -> Network ，展开Outbound requests

   Collapse，勾选第一项即可。再次test push events，成功。

   ![](images\gitlab-webhook-success.jpg)

8. 配置free项目，增加构建步骤，执行shell，将发送钉钉消息的shell保存

9. 提交代码到gitlab仓库，查看构建是否自动执行