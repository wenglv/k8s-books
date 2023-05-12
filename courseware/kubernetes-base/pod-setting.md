###### Pod数据持久化

若删除了Pod，由于mysql的数据都在容器内部，会造成数据丢失，因此需要数据进行持久化。

- 定点使用hostpath挂载，nodeSelector定点

  `myblog/one-pod/pod-with-volume.yaml`

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: myblog
    namespace: luffy
    labels:
      component: myblog
  spec:
    volumes: 
    - name: mysql-data
      hostPath: 
        path: /opt/mysql/data
    nodeSelector:   # 使用节点选择器将Pod调度到指定label的节点
      component: mysql
    containers:
    - name: myblog
      image: 172.21.51.143:5000/myblog:v1
      env:
      - name: MYSQL_HOST   #  指定root用户的用户名
        value: "127.0.0.1"
      - name: MYSQL_PASSWD
        value: "123456"
      ports:
      - containerPort: 8002
    - name: mysql
      image: mysql:5.7
      args:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      ports:
      - containerPort: 3306
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "123456"
      - name: MYSQL_DATABASE
        value: "myblog"
      volumeMounts:
      - name: mysql-data
        mountPath: /var/lib/mysql
  
  
  ```

  保存文件为`pod-with-volume.yaml`，执行创建

  ```bash
  ## 若存在旧的同名服务，先删除掉，后创建
  $ kubectl -n luffy delete pod myblog
  ## 创建
  $ kubectl create -f pod-with-volume.yaml
  
  ## 此时pod状态Pending
  $ kubectl -n luffy get po
  NAME     READY   STATUS    RESTARTS   AGE
  myblog   0/2     Pending   0          32s
  
  ## 查看原因，提示调度失败，因为节点不满足node selector
  $ kubectl -n luffy describe po myblog
  Events:
    Type     Reason            Age                From               Message
    ----     ------            ----               ----               -------
    Warning  FailedScheduling  12s (x2 over 12s)  default-scheduler  0/3 nodes are available: 3 node(s) didn't match node selector.
    
  ## 为节点打标签
  $ kubectl label node k8s-slave1 component=mysql
  
  ## 再次查看，已经运行成功
  $ kubectl -n luffy get po
  NAME     READY   STATUS    RESTARTS   AGE     IP             NODE
  myblog   2/2     Running   0          3m54s   10.244.1.150   k8s-slave1
  
  ## 到k8s-slave1节点，查看/opt/mysql/data
  $ ll /opt/mysql/data/
  total 188484
  -rw-r----- 1 polkitd input       56 Mar 29 09:20 auto.cnf
  -rw------- 1 polkitd input     1676 Mar 29 09:20 ca-key.pem
  -rw-r--r-- 1 polkitd input     1112 Mar 29 09:20 ca.pem
  drwxr-x--- 2 polkitd input     8192 Mar 29 09:20 sys
  ...
  
  ## 执行migrate，创建数据库表，然后删掉pod，再次创建后验证数据是否存在
  $ kubectl -n luffy exec -ti myblog python3 manage.py migrate
  
  ## 访问服务，正常
  $ curl 10.244.1.150:8002/blog/index/ 
  
  ## 删除pod
  $ kubectl delete -f pod-with-volume.yaml
  
  ## 再次创建Pod
  $ kubectl create -f pod-with-volume.yaml
  
  ## 查看pod ip并访问服务
  $ kubectl -n luffy get po -o wide
  NAME     READY   STATUS    RESTARTS   AGE   IP             NODE  
  myblog   2/2     Running   0          7s    10.244.1.151   k8s-slave1
  
  ## 未重新做migrate，服务正常
  $ curl 10.244.1.151:8002/blog/index/
  
  ```

- 使用PV+PVC连接分布式存储解决方案

  - ceph
  - glusterfs
  - nfs

###### 服务健康检查

检测容器服务是否健康的手段，若不健康，会根据设置的重启策略（restartPolicy）进行操作，两种检测机制可以分别单独设置，若不设置，默认认为Pod是健康的。

两种机制：

- LivenessProbe探针
  存活性探测：用于判断容器是否存活，即Pod是否为running状态，如果LivenessProbe探针探测到容器不健康，则kubelet将kill掉容器，并根据容器的重启策略是否重启，如果一个容器不包含LivenessProbe探针，则Kubelet认为容器的LivenessProbe探针的返回值永远成功。 

  ```yaml
  ...
    containers:
    - name: myblog
      image: 172.21.51.143:5000/myblog:v1
      livenessProbe:
        httpGet:
          path: /blog/index/
          port: 8002
          scheme: HTTP
        initialDelaySeconds: 10  # 容器启动后第一次执行探测是需要等待多少秒
        periodSeconds: 10 	# 执行探测的频率
        timeoutSeconds: 2		# 探测超时时间
  ...
  ```

  

  ![](images\livenessprobe.webp)

- ReadinessProbe探针
  可用性探测：用于判断容器是否正常提供服务，即容器的Ready是否为True，是否可以接收请求，如果ReadinessProbe探测失败，则容器的Ready将为False， Endpoint Controller 控制器将此Pod的Endpoint从对应的service的Endpoint列表中移除，不再将任何请求调度此Pod上，直到下次探测成功。（剔除此pod不参与接收请求不会将流量转发给此Pod）。

  ```yaml
  ...
    containers:
    - name: myblog
      image: 172.21.51.143:5000/myblog:v1
      readinessProbe: 
        httpGet: 
          path: /blog/index/
          port: 8002
          scheme: HTTP
        initialDelaySeconds: 10 
        timeoutSeconds: 2
        periodSeconds: 10
  ...
  
  ```

  

  ![](images\readnessprobe.webp)

三种类型：

- exec：通过执行命令来检查服务是否正常，返回值为0则表示容器健康

  ```bash
  ...
      livenessProbe:
        exec:
          command:
          - cat
          - /tmp/healthy
        initialDelaySeconds: 5
        periodSeconds: 5
  ...
  ```

  

- httpGet方式：通过发送http请求检查服务是否正常，返回200-399状态码则表明容器健康

- tcpSocket：通过容器的IP和Port执行TCP检查，如果能够建立TCP连接，则表明容器健康

  ```bash
    ...
        livenessProbe:
          tcpSocket:
            port: 8002
          initialDelaySeconds: 10  # 容器启动后第一次执行探测是需要等待多少秒
          periodSeconds: 10     # 执行探测的频率
          timeoutSeconds: 2        # 探测超时时间
    ...
  ```

示例：

完整文件路径 `myblog/one-pod/pod-with-healthcheck.yaml`

```yaml
  containers:
  - name: myblog
    image: 172.21.51.143:5000/myblog:v1
    env:
    - name: MYSQL_HOST   #  指定root用户的用户名
      value: "127.0.0.1"
    - name: MYSQL_PASSWD
      value: "123456"
    ports:
    - containerPort: 8002
    livenessProbe:
      httpGet:
        path: /blog/index/
        port: 8002
        scheme: HTTP
      initialDelaySeconds: 10  # 容器启动后第一次执行探测是需要等待多少秒
      periodSeconds: 10 	# 执行探测的频率
      timeoutSeconds: 2		# 探测超时时间
    readinessProbe: 
      httpGet: 
        path: /blog/index/
        port: 8002
        scheme: HTTP
      initialDelaySeconds: 10 
      timeoutSeconds: 2
      periodSeconds: 10


```

- initialDelaySeconds：容器启动后第一次执行探测是需要等待多少秒。
- periodSeconds：执行探测的频率。默认是10秒，最小1秒。
- timeoutSeconds：探测超时时间。默认1秒，最小1秒。
- successThreshold：探测失败后，最少连续探测成功多少次才被认定为成功。默认是1。
- failureThreshold：探测成功后，最少连续探测失败多少次
  才被认定为失败。默认是3，最小值是1。

K8S将在Pod开始**启动10s(initialDelaySeconds)后**利用HTTP访问8002端口的/blog/index/，如果**超过2s**或者返回码不在200~399内，则健康检查失败

###### 重启策略

Pod的重启策略（RestartPolicy）应用于Pod内的所有容器，并且仅在Pod所处的Node上由kubelet进行判断和重启操作。当某个容器异常退出或者健康检查失败时，kubelet将根据RestartPolicy的设置来进行相应的操作。
 Pod的重启策略包括Always、OnFailure和Never，默认值为Always。

- Always：当容器进程退出后，由kubelet自动重启该容器；
- OnFailure：当容器终止运行且退出码不为0时，由kubelet自动重启该容器；
- Never：不论容器运行状态如何，kubelet都不会重启该容器。

演示重启策略：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-restart-policy
spec:
  restartPolicy: Always
  containers:
  - name: busybox
    image: busybox
    args:
    - /bin/sh
    - -c
    - sleep 10

```

1. 使用默认的重启策略，即 restartPolicy: Always ，无论容器是否是正常退出，都会自动重启容器
2. 使用OnFailure的策略时
   - 如果把exit 1，去掉，即让容器的进程正常退出的话，则不会重启
   - 只有非正常退出状态才会重启
3. 使用Never时，退出了就不再重启

可以看出，若容器正常退出，Pod的状态会是Completed，非正常退出，状态为CrashLoopBackOff

###### 镜像拉取策略

```yaml
spec:
  containers:
  - name: myblog
    image: 172.21.51.143:5000/myblog:v1
    imagePullPolicy: IfNotPresent


```

设置镜像的拉取策略，默认为IfNotPresent

- Always，总是拉取镜像，即使本地有镜像也从仓库拉取
- IfNotPresent ，本地有则使用本地镜像，本地没有则去仓库拉取
- Never，只使用本地镜像，本地没有则报错

###### Pod资源限制

为了保证充分利用集群资源，且确保重要容器在运行周期内能够分配到足够的资源稳定运行，因此平台需要具备

Pod的资源限制的能力。 对于一个pod来说，资源最基础的2个的指标就是：CPU和内存。

Kubernetes提供了个采用requests和limits 两种类型参数对资源进行预分配和使用限制。

完整文件路径：`myblog/one-pod/pod-with-resourcelimits.yaml`

```yaml
...
  containers:
  - name: myblog
    image: 172.21.51.143:5000/myblog:v1
    env:
    - name: MYSQL_HOST   #  指定root用户的用户名
      value: "127.0.0.1"
    - name: MYSQL_PASSWD
      value: "123456"
    ports:
    - containerPort: 8002
    resources:
      requests:
        memory: 100Mi
        cpu: 50m
      limits:
        memory: 500Mi
        cpu: 100m
...


```

requests：

- 容器使用的最小资源需求,作用于schedule阶段，作为容器调度时资源分配的判断依赖
- 只有当前节点上可分配的资源量 >= request 时才允许将容器调度到该节点
- request参数不限制容器的最大可使用资源
- requests.cpu被转成docker的--cpu-shares参数，与cgroup cpu.shares功能相同 (无论宿主机有多少个cpu或者内核，--cpu-shares选项都会按照比例分配cpu资源）
- requests.memory没有对应的docker参数，仅作为k8s调度依据

limits：

- 容器能使用资源的最大值
- 设置为0表示对使用的资源不做限制, 可无限的使用
- 当pod 内存超过limit时，会被oom
- 当cpu超过limit时，不会被kill，但是会限制不超过limit值
- limits.cpu会被转换成docker的–cpu-quota参数。与cgroup cpu.cfs_quota_us功能相同
- limits.memory会被转换成docker的–memory参数。用来限制容器使用的最大内存

 对于 CPU，我们知道计算机里 CPU 的资源是按`“时间片”`的方式来进行分配的，系统里的每一个操作都需要 CPU 的处理，所以，哪个任务要是申请的 CPU 时间片越多，那么它得到的 CPU 资源就越多。

然后还需要了解下 CGroup 里面对于 CPU 资源的单位换算：

```shell
1 CPU =  1000 millicpu（1 Core = 1000m）


```

 这里的 `m` 就是毫、毫核的意思，Kubernetes 集群中的每一个节点可以通过操作系统的命令来确认本节点的 CPU 内核数量，然后将这个数量乘以1000，得到的就是节点总 CPU 总毫数。比如一个节点有四核，那么该节点的 CPU 总毫量为 4000m。 

`docker run`命令和 CPU 限制相关的所有选项如下：

| 选项                  | 描述                                                    |
| --------------------- | ------------------------------------------------------- |
| `--cpuset-cpus=""`    | 允许使用的 CPU 集，值可以为 0-3,0,1                     |
| `-c`,`--cpu-shares=0` | CPU 共享权值（相对权重）                                |
| `cpu-period=0`        | 限制 CPU CFS 的周期，范围从 100ms~1s，即[1000, 1000000] |
| `--cpu-quota=0`       | 限制 CPU CFS 配额，必须不小于1ms，即 >= 1000，绝对限制  |

```shell
docker run -it --cpu-period=50000 --cpu-quota=25000 ubuntu:16.04 /bin/bash


```

将 CFS 调度的周期设为 50000，将容器在每个周期内的 CPU 配额设置为 25000，表示该容器每 50ms 可以得到 50% 的 CPU 运行时间。

> 注意：若内存使用超出限制，会引发系统的OOM机制，因CPU是可压缩资源，不会引发Pod退出或重建

