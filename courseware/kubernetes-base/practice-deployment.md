###### Deployment

`myblog/deployment/deploy-mysql.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: luffy
spec:
  replicas: 1	#指定Pod副本数
  selector:		#指定Pod的选择器
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:	#给Pod打label
        app: mysql
    spec:
      hostNetwork: true
      volumes: 
      - name: mysql-data
        hostPath: 
          path: /opt/mysql/data
      nodeSelector:   # 使用节点选择器将Pod调度到指定label的节点
        component: mysql
      containers:
      - name: mysql
        image: mysql:5.7
        args:
        - --character-set-server=utf8mb4
        - --collation-server=utf8mb4_unicode_ci
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: myblog
              key: MYSQL_USER
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myblog
              key: MYSQL_PASSWD
        - name: MYSQL_DATABASE
          value: "myblog"
        resources:
          requests:
            memory: 100Mi
            cpu: 50m
          limits:
            memory: 500Mi
            cpu: 100m
        readinessProbe:
          tcpSocket:
            port: 3306
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          tcpSocket:
            port: 3306
          initialDelaySeconds: 15
          periodSeconds: 20
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql

```

deploy-myblog.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myblog
  namespace: luffy
spec:
  replicas: 1	#指定Pod副本数
  selector:		#指定Pod的选择器
    matchLabels:
      app: myblog
  template:
    metadata:
      labels:	#给Pod打label
        app: myblog
    spec:
      containers:
      - name: myblog
        image: 172.21.51.143:5000/myblog:v1
        imagePullPolicy: IfNotPresent
        env:
        - name: MYSQL_HOST
          valueFrom:
            configMapKeyRef:
              name: myblog
              key: MYSQL_HOST
        - name: MYSQL_PORT
          valueFrom:
            configMapKeyRef:
              name: myblog
              key: MYSQL_PORT
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: myblog
              key: MYSQL_USER
        - name: MYSQL_PASSWD
          valueFrom:
            secretKeyRef:
              name: myblog
              key: MYSQL_PASSWD
        ports:
        - containerPort: 8002
        resources:
          requests:
            memory: 100Mi
            cpu: 50m
          limits:
            memory: 500Mi
            cpu: 100m
        livenessProbe:
          httpGet:
            path: /blog/index/
            port: 8002
            scheme: HTTP
          initialDelaySeconds: 10  # 容器启动后第一次执行探测是需要等待多少秒
          periodSeconds: 15 	# 执行探测的频率
          timeoutSeconds: 2		# 探测超时时间
        readinessProbe: 
          httpGet: 
            path: /blog/index/
            port: 8002
            scheme: HTTP
          initialDelaySeconds: 10 
          timeoutSeconds: 2
          periodSeconds: 15

```



###### 创建Deployment

```bash
$ kubectl create -f deploy.yaml


```

###### 查看Deployment

```bash
# kubectl api-resources
$ kubectl -n luffy get deploy
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
myblog   1/1     1            1           2m22s
mysql    1/1     1            1           2d11h

  * `NAME` 列出了集群中 Deployments 的名称。
  * `READY`显示当前正在运行的副本数/期望的副本数。
  * `UP-TO-DATE`显示已更新以实现期望状态的副本数。
  * `AVAILABLE`显示应用程序可供用户使用的副本数。
  * `AGE` 显示应用程序运行的时间量。

# 查看pod
$ kubectl -n luffy get po
NAME                      READY   STATUS    RESTARTS   AGE
myblog-7c96c9f76b-qbbg7   1/1     Running   0          109s
mysql-85f4f65f99-w6jkj    1/1     Running   0          2m28s

# 查看replicaSet
$ kubectl -n luffy get rs

```



###### 副本保障机制

controller实时检测pod状态，并保障副本数一直处于期望的值。

```bash
## 删除pod，观察pod状态变化
$ kubectl -n luffy delete pod myblog-7c96c9f76b-qbbg7

# 观察pod
$ kubectl get pods -o wide

## 设置两个副本, 或者通过kubectl -n luffy edit deploy myblog的方式，最好通过修改文件，然后apply的方式，这样yaml文件可以保持同步
$ kubectl -n luffy scale deploy myblog --replicas=2
deployment.extensions/myblog scaled

# 观察pod
$ kubectl get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE
myblog-7c96c9f76b-qbbg7   1/1     Running   0          11m
myblog-7c96c9f76b-s6brm   1/1     Running   0          55s
mysql-85f4f65f99-w6jkj    1/1     Running   0          11m


```

###### Pod驱逐策略

K8S 有个特色功能叫 pod eviction，它在某些场景下如节点 NotReady，或者资源不足时，把 pod 驱逐至其它节点，这也是出于业务保护的角度去考虑的。

1. Kube-controller-manager: 周期性检查所有节点状态，当节点处于 NotReady 状态超过一段时间后，驱逐该节点上所有 pod。

   - `pod-eviction-timeout`：NotReady 状态节点超过该时间后，执行驱逐，默认 5 min，适用于k8s 1.13版本之前

   - 1.13版本后，集群开启` TaintBasedEvictions 与TaintNodesByCondition` 功能，即[taint-based-evictions](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)，即节点若失联或者出现各种异常情况，k8s会自动为node打上污点，同时为pod默认添加如下容忍设置：

     ```yaml
       tolerations:
       - effect: NoExecute
         key: node.kubernetes.io/not-ready
         operator: Exists
         tolerationSeconds: 300
       - effect: NoExecute
         key: node.kubernetes.io/unreachable
         operator: Exists
         tolerationSeconds: 300
     ```

     即各pod可以独立设置驱逐容忍时间。

2. Kubelet: 周期性检查本节点资源，当资源不足时，按照优先级驱逐部分 pod

   - `memory.available`：节点可用内存
   - `nodefs.available`：节点根盘可用存储空间
   - `nodefs.inodesFree`：节点inodes可用数量
   - `imagefs.available`：镜像存储盘的可用空间
   - `imagefs.inodesFree`：镜像存储盘的inodes可用数量



###### 服务更新

修改服务，重新打tag模拟服务更新。

更新方式：

1. 修改yaml文件，使用`kubectl -n luffy apply -f deploy-myblog.yaml`来应用更新

2. `kubectl -n luffy edit deploy myblog`在线更新

3. `kubectl -n luffy set image deploy myblog myblog=172.21.51.143:5000/myblog:v2 --record` 

修改文件测试：

```bash
$ vi mybolg/blog/template/index.html

$ docker build . -t 172.21.51.143:5000/myblog:v2 -f Dockerfile
$ docker push 172.21.51.143:5000/myblog:v2


```

