###### 更新策略

```yaml
...
spec:
  replicas: 2	#指定Pod副本数
  selector:		#指定Pod的选择器
    matchLabels:
      app: myblog
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate		#指定更新方式为滚动更新，默认策略，通过get deploy yaml查看
    ...

```

![](images/update.png)

策略控制：

- maxSurge：最大激增数, 指更新过程中, 最多可以比replicas预先设定值多出的pod数量, 可以为固定值或百分比,默认为desired Pods数的25%。计算时向上取整(比如3.4，取4)，更新过程中最多会有replicas + maxSurge个pod
- maxUnavailable： 指更新过程中, 最多有几个pod处于无法服务状态 , 可以为固定值或百分比，默认为desired Pods数的25%。计算时向下取整(比如3.6，取3)

*在Deployment rollout时，需要保证Available(Ready) Pods数不低于 desired pods number - maxUnavailable; 保证所有的非异常状态Pods数不多于 desired pods number + maxSurge*。

以myblog为例，使用默认的策略，更新过程:

1. maxSurge 25%，2个实例，向上取整，则maxSurge为1，意味着最多可以有2+1=3个Pod，那么此时会新创建1个ReplicaSet，RS-new，把副本数置为1，此时呢，副本控制器就去创建这个新的Pod
2. 同时，maxUnavailable是25%，副本数2*25%，向下取整，则为0，意味着，滚动更新的过程中，不能有少于2个可用的Pod，因此，旧的Replica（RS-old）会先保持不动，等RS-new管理的Pod状态Ready后，此时已经有3个Ready状态的Pod了，那么由于只要保证有2个可用的Pod即可，因此，RS-old的副本数会有2个变成1个，此时，会删掉一个旧的Pod
3. 删掉旧的Pod的时候，由于总的Pod数量又变成2个了，因此，距离最大的3个还有1个Pod可以创建，所以，RS-new把管理的副本数由1改成2，此时又会创建1个新的Pod，等RS-new管理了2个Pod都ready后，那么就可以把RS-old的副本数由1置为0了，这样就完成了滚动更新

```bash
#查看滚动更新事件
$ kubectl -n luffy describe deploy myblog
...
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  11s   deployment-controller  Scaled up replica set myblog-6cf56fc848 to 1
  Normal  ScalingReplicaSet  11s   deployment-controller  Scaled down replica set myblog-6fdcf98f9 to 1
  Normal  ScalingReplicaSet  11s   deployment-controller  Scaled up replica set myblog-6cf56fc848 to 2
  Normal  ScalingReplicaSet  6s    deployment-controller  Scaled down replica set myblog-6fdcf98f9 to 0
$ kubectl get rs
NAME                     DESIRED   CURRENT   READY   AGE
myblog-6cf56fc848   2         2         2       16h
myblog-6fdcf98f9    0         0         0       16h


```

除了滚动更新以外，还有一种策略是Recreate，直接在当前的pod基础上先删后建：

```bash
...
  strategy:
    type: Recreate
...
```

我们课程中的mysql服务应该使用Recreate来管理：

```bash
$ kubectl -n luffy edit deploy mysql
...
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: mysql
...
```



###### 服务回滚

通过滚动升级的策略可以平滑的升级Deployment，若升级出现问题，需要最快且最好的方式回退到上一次能够提供正常工作的版本。为此K8S提供了回滚机制。

**revision**：更新应用时，K8S都会记录当前的版本号，即为revision，当升级出现问题时，可通过回滚到某个特定的revision，默认配置下，K8S只会保留最近的几个revision，可以通过Deployment配置文件中的spec.revisionHistoryLimit属性增加revision数量，默认是10。

查看当前：

```bash
$ kubectl -n luffy rollout history deploy myblog ##CHANGE-CAUSE为空
$ kubectl delete -f deploy-myblog.yaml    ## 方便演示到具体效果，删掉已有deployment


```

记录回滚：

```bash
$ kubectl apply -f deploy-myblog.yaml --record

$ kubectl -n luffy set image deploy myblog myblog=172.21.51.143:5000/myblog:v2 --record=true


```

查看deployment更新历史：

```bash
$ kubectl -n luffy rollout history deploy myblog
deployment.extensions/myblog
REVISION  CHANGE-CAUSE
1         kubectl create --filename=deploy-myblog.yaml --record=true
2         kubectl set image deploy myblog myblog=172.21.51.143:5000/demo/myblog:v1 --record=true


```

回滚到具体的REVISION:

```bash
$ kubectl -n luffy rollout undo deploy myblog --to-revision=1
deployment.extensions/myblog rolled back

# 访问应用测试


```

