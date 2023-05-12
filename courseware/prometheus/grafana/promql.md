###### Metrics指标类型与PromQL

TSDB的样本分布示意图：

```bash
  ^
  │   . . . . . . . . . . . . . . . . .   . .   node_cpu{cpu="cpu0",mode="idle"}
  │     . . . . . . . . . . . . . . . . . . .   node_cpu{cpu="cpu0",mode="system"}
  │     . . . . . . . . . .   . . . . . . . .   node_load1{}
  │     . . . . . . . . . . . . . . . .   . .   node_cpu_seconds_total{...}
  v
    <------------------ 时间 ---------------->
```

Guage类型：

```bash
$ kubectl -n monitor get po -o wide |grep k8s-master
node-exporter-ld6sq    1/1     Running   0          4d3h    172.21.51.143   k8s-master
$ curl -s  172.21.51.143:9100/metrics |grep node_load1
# HELP node_load1 1m load average.
# TYPE node_load1 gauge
node_load1 0.18
# HELP node_load15 15m load average.
# TYPE node_load15 gauge
node_load15 0.37
```

Gauge类型的指标侧重于反应系统的当前状态。

- 这类指标的样本数据可增可减。
- 常见指标如：node_memory_MemAvailable_bytes（可用内存大小）、node_load1（系统平均负载）

Guage类型的数据，通常直接查询就会有比较直观的业务含义，比如：

- node_load5
- node_memory_MemAvailable_bytes

我们也会对这类数据做简单的处理，比如：

- 过滤其中某些节点
- 对指标进行数学运算

这就是PromQL提供的能力，可以对收集到的数据做聚合、计算等处理。

PromQL（ Prometheus Query Language ）是Prometheus自定义的一套强大的数据查询语言，除了使用监控指标作为查询关键字以为，还内置了大量的函数，帮助用户进一步对时序数据进行处理。

比如：

- 只显示k8s-master节点的平均负载

  ```bash
  node_load1{instance="k8s-master"}
  ```

- 显示除了k8s-master节点外的其他节点的平均负载

  ```bash
  node_load1{instance!="k8s-master"}
  ```

- 正则匹配

  ```bash
  node_load1{instance=~"k8s-master|k8s-slave1"}
  ```

- 集群各节点系统内存使用率

  ```bash
 ((node_memory_MemTotal_bytes - (node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes))/node_memory_MemTotal_bytes)*100
  ```



counter类型：

```bash
$ curl -s  172.21.51.67:9100/metrics |grep node_cpu_seconds_total
# HELP node_cpu_seconds_total Seconds the cpus spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 294341.02
node_cpu_seconds_total{cpu="0",mode="iowait"} 120.78
node_cpu_seconds_total{cpu="0",mode="irq"} 0
node_cpu_seconds_total{cpu="0",mode="nice"} 0.13
node_cpu_seconds_total{cpu="0",mode="softirq"} 1263.29
```



counter类型的指标其工作方式和计数器一样，只增不减（除非系统发生重置）。常见的监控指标，如http_requests_total，node_cpu_seconds_total都是Counter类型的监控指标。 

通常计数器类型的指标，名称后面都以`_total`结尾。我们通过理解CPU利用率的PromQL表达式来讲解Counter指标类型的使用。

各节点CPU的平均使用率表达式：

```bash
(1- sum(increase(node_cpu_seconds_total{mode="idle"}[2m])) by (instance) / sum(increase(node_cpu_seconds_total{}[2m])) by (instance)) * 100
```



分析：

`node_cpu_seconds_total`的指标含义是统计系统运行以来，CPU资源分配的时间总数，单位为秒，是累加的值。比如，直接运行该指标：

```bash
node_cpu_seconds_total
# 显示的是所有节点、所有CPU核心、在各种工作模式下分配的时间总和
```

其中mode的值和我们平常在系统中执行`top`命令看到的CPU显示的信息一致：

每个mode对应的含义如下：

- `user`(us)
  表示用户态空间或者说是用户进程(running user space processes)使用CPU所耗费的时间。这是日常我们部署的应用所在的层面，最常见常用。
- `system`(sy)
  表示内核态层级使用CPU所耗费的时间。分配内存、IO操作、创建子进程……都是内核操作。这也表明，当IO操作频繁时，System参数会很高。
- `steal`(st)
  当运行在虚拟化环境中，花费在其它 OS 中的时间（基于虚拟机监视器 hypervisor 的调度）；可以理解成由于虚拟机调度器将 cpu 时间用于其它 OS 了，故当前 OS 无法使用 CPU 的时间。
- `softirq`(si)
  从系统启动开始，累计到当前时刻，软中断时间
- `irq`(hi)
  从系统启动开始，累计到当前时刻，硬中断时间
- `nice`(ni)
  从系统启动开始，累计到当前时刻， 低优先级(低优先级意味着进程 nice 值小于 0)用户态的进程所占用的CPU时间
- `iowait`(wa)
  从系统启动开始，累计到当前时刻，IO等待时间
- `idle`(id)
  从系统启动开始，累计到当前时刻，除IO等待时间以外的其它等待时间，亦即空闲时间



我们通过指标拿到的各核心cpu分配的总时长数据，都是瞬时的数据，如何转换成 CPU的利用率？

先来考虑如何我们如何计算CPU利用率，假如我的k8s-master节点是4核CPU，我们来考虑如下场景：

- 过去60秒内每个CPU核心处于idle空闲状态的时长，假如分别为 :
  - cpu0：20s
  - cpu1：30s
  - cpu2：50s
  - cpu3：40s
- 则四个核心总共可分配的时长是 4*60=240s
- 实际空闲状态的总时长为20+30+50+40=140s
- 那么我们可以计算出过去1分钟k8s-master节点的CPU利用率为 (1- 140/240) * 100 = 41.7%

因此，我们只需要使用PromQL取出上述过程中的值即可：

```bash
# 过滤出当前时间点idle的时长
node_cpu_seconds_total{mode="idle"}

# 使用[1m]取出1分钟区间内的样本值,注意，1m区间要大于prometheus设置的抓取周期，此处会将周期内所以的样本值取出
node_cpu_seconds_total{mode="idle"}[1m]

# 使用increase方法，获取该区间内idle状态的增量值,即1分钟内，mode="idle"状态增加的时长
increase(node_cpu_seconds_total{mode="idle"}[1m])

# 由于是多个cpu核心，因此需要做累加，使用sum函数
sum(increase(node_cpu_seconds_total{mode="idle"}[1m]))

# 由于是多台机器，因此，需要按照instance的值进行分组累加，使用by关键字做分组,这样就获得了1分钟内，每个节点上 所有CPU核心idle状态的增量时长，即前面示例中的”20+30+50+40=140s“
sum(increase(node_cpu_seconds_total{mode="idle"}[1m])) by (instance)

# 去掉mode=idle的过滤条件，即可获取1分钟内，所有状态的cpu获得的增量总时长，即4*60=240s
sum(increase(node_cpu_seconds_total{}[1m])) by (instance)

# 最终的语句
(1- sum(increase(node_cpu_seconds_total{mode="idle"}[1m])) by (instance) / sum(increase(node_cpu_seconds_total{}[1m])) by (instance)) * 100
```



除此之外，还会经常看到avg,irate和rate方法的使用：

 `irate()` 是基于最后两个数据点计算一个时序指标在一个范围内的每秒递增率 ，举个例子：

```bash
# 1min内，k8s-master节点的idle状态的cpu分配时长增量值
increase(node_cpu_seconds_total{instance="k8s-master",mode="idle"}[1m])

{cpu="0",instance="k8s-master",job="kubernetes-sd-node-exporter",mode="idle"}	56.5
{cpu="1",instance="k8s-master",job="kubernetes-sd-node-exporter",mode="idle"}	56.04
{cpu="2",instance="k8s-master",job="kubernetes-sd-node-exporter",mode="idle"}	56.6
{cpu="3",instance="k8s-master",job="kubernetes-sd-node-exporter",mode="idle"}	56.5

#以第一条数据为例，说明过去的1分钟，k8s-master节点的第一个CPU核心，有56.5秒的时长是出于idle状态的

# 1min内，k8s-master节点的idle状态的cpu分配每秒的速率
irate(node_cpu_seconds_total{instance="k8s-master",mode="idle"}[1m])
{cpu="0",instance="k8s-master",job="kubernetes-sd-node-exporter",mode="idle"}	0.934
{cpu="1",instance="k8s-master",job="kubernetes-sd-node-exporter",mode="idle"}	0.932
{cpu="2",instance="k8s-master",job="kubernetes-sd-node-exporter",mode="idle"}	0.933
{cpu="3",instance="k8s-master",job="kubernetes-sd-node-exporter",mode="idle"}	0.936
# 该值如何计算的？
# irate会取出样本中的最后两个点来作为增长依据，然后做差值计算，并且除以两个样本间的数据时长，也就是说，我们设置2m,5m取出来的值是一样的，因为只会计算最后两个样本差。
# 以第一条数据为例，表示用irate计算出来的结果是，过去的两分钟内，cpu平均每秒钟有0.934秒的时间是处于idle状态的


# rate会1min内第一个和最后一个样本值为依据，计算方式和irate保持一致
rate(node_cpu_seconds_total{instance="k8s-master",mode="idle"}[1m])
{cpu="0",instance="k8s-master",job="kubernetes-sd-node-exporter",mode="idle"}	0.933
{cpu="1",instance="k8s-master",job="kubernetes-sd-node-exporter",mode="idle"}	0.940
{cpu="2",instance="k8s-master",job="kubernetes-sd-node-exporter",mode="idle"}	0.935
{cpu="3",instance="k8s-master",job="kubernetes-sd-node-exporter",mode="idle"}	0.937
```

因此rate的值，相对来讲更平滑，因为计算的是时间段内的平均，更适合于用作告警。

取CPU平均使用率也可以用如下表达式表示：

`(1 - avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) by (instance)) * 100`

