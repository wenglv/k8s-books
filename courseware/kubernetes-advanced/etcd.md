#### ETCD常用操作

拷贝etcdctl命令行工具：

```bash
$ docker cp etcd_container:/usr/local/bin/etcdctl /usr/bin/etcdctl
```

查看etcd集群的成员节点：

```bash
$ export ETCDCTL_API=3
$ etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key member list -w table

$ alias etcdctl='etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key'

$ etcdctl member list -w table
```

查看etcd集群节点状态：

```bash
$ etcdctl endpoint status -w table

$ etcdctl endpoint health -w table
```



设置key值:

```bash
$ etcdctl put luffy 1
$ etcdctl get luffy
```



查看所有key值：

```bash
$  etcdctl get / --prefix --keys-only

```

查看具体的key对应的数据：

```bash
$ etcdctl get /registry/pods/jenkins/sonar-postgres-7fc5d748b6-gtmsb

```

list-watch:

```bash
$ etcdctl watch /luffy/ --prefix
$ etcdctl put /luffy/key1 val1
```



添加定时任务做数据快照（重要！）

```bash
$ etcdctl snapshot save `hostname`-etcd_`date +%Y%m%d%H%M`.db
```

恢复快照：

1. 停止etcd和apiserver

2. 移走当前数据目录

   ```bash
   $ mv /var/lib/etcd/ /tmp
   ```

3. 恢复快照

   ```bash
   $ etcdctl snapshot restore `hostname`-etcd_`date +%Y%m%d%H%M`.db --data-dir=/var/lib/etcd/
   
   ```

4. 集群恢复

   https://github.com/etcd-io/etcd/blob/release-3.3/Documentation/op-guide/recovery.md



- namespace删除问题

  很多情况下，会出现namespace删除卡住的问题，此时可以通过操作etcd来删除数据：

  

