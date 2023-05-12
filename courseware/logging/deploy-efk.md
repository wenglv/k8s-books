##### 部署es服务

###### 部署分析

1. es生产环境是部署es集群，通常会使用statefulset进行部署
2. es默认使用elasticsearch用户启动进程，es的数据目录是通过宿主机的路径挂载，因此目录权限被主机的目录权限覆盖，因此可以利用initContainer容器在es进程启动之前把目录的权限修改掉，注意init container要用特权模式启动。
3. 若希望使用helm部署，参考 https://github.com/helm/charts/tree/master/stable/elasticsearch 

https://staight.github.io/2019/09/16/%E5%9C%A8k8s%E4%B8%8A%E9%83%A8%E7%BD%B2elasticsearch/

###### 使用StatefulSet管理有状态服务

使用Deployment创建多副本的pod的情况：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
  labels:
    app: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-deployment
  template:
    metadata:
      labels:
        app: nginx-deployment
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```



使用StatefulSet创建多副本pod的情况：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-statefulset
  namespace: default
  labels:
    app: nginx-sts
spec:
  replicas: 3
  serviceName: "nginx"
  selector:
    matchLabels:
      app: nginx-sts
  template:
    metadata:
      labels:
        app: nginx-sts
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```

无头服务Headless Service

```yaml
kind: Service
apiVersion: v1
metadata:
  name: nginx
  namespace: default
spec:
  selector:
    app: nginx-sts
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  clusterIP: None
```

```bash
$ kubectl -n default exec  -ti nginx-statefulset-0 sh
/ # curl nginx-statefulset-2.nginx
```



###### 部署并验证

`es-config.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: es-config
  namespace: logging
data:
  elasticsearch.yml: |
    cluster.name: "luffy-elasticsearch"
    node.name: "${POD_NAME}"
    network.host: 0.0.0.0
    discovery.seed_hosts: "es-svc-headless"
    cluster.initial_master_nodes: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
```

`es-svc-headless.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: es-svc-headless
  namespace: logging
  labels:
    k8s-app: elasticsearch
spec:
  selector:
    k8s-app: elasticsearch
  clusterIP: None
  ports:
  - name: in
    port: 9300
    protocol: TCP
```

`es-statefulset.yaml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    k8s-app: elasticsearch
spec:
  replicas: 3
  serviceName: es-svc-headless
  selector:
    matchLabels:
      k8s-app: elasticsearch
  template:
    metadata:
      labels:
        k8s-app: elasticsearch
    spec:
      initContainers:
      - command:
        - /sbin/sysctl
        - -w
        - vm.max_map_count=262144
        image: alpine:3.6
        imagePullPolicy: IfNotPresent
        name: elasticsearch-logging-init
        resources: {}
        securityContext:
          privileged: true
      - name: fix-permissions
        image: alpine:3.6
        command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: es-data-volume
          mountPath: /usr/share/elasticsearch/data
      containers:
      - name: elasticsearch
        image: elasticsearch:7.4.2
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
        resources:
          limits:
            cpu: '1'
            memory: 2Gi
          requests:
            cpu: '1'
            memory: 2Gi
        ports:
        - containerPort: 9200
          name: db
          protocol: TCP
        - containerPort: 9300
          name: transport
          protocol: TCP
        volumeMounts:
          - name: es-config-volume
            mountPath: /usr/share/elasticsearch/config/elasticsearch.yml
            subPath: elasticsearch.yml
          - name: es-data-volume
            mountPath: /usr/share/elasticsearch/data
      volumes:
        - name: es-config-volume
          configMap:
            name: es-config
            items:
            - key: elasticsearch.yml
              path: elasticsearch.yml
  volumeClaimTemplates:
  - metadata:
      name: es-data-volume
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "nfs"
      resources:
        requests:
          storage: 5Gi
```

`es-svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: es-svc
  namespace: logging
  labels:
    k8s-app: elasticsearch
spec:
  selector:
    k8s-app: elasticsearch
  ports:
  - name: out
    port: 9200
    protocol: TCP
```





```bash
$ kubectl create namespace logging

## 部署服务
$ kubectl apply -f es-config.yaml
$ kubectl apply -f es-svc-headless.yaml
$ kubectl apply -f es-sts.yaml
$ kubectl apply -f es-svc.yaml

## 等待片刻，查看一下es的pod部署到了k8s-slave1节点，状态变为running
$ kubectl -n logging get po -o wide  
NAME              READY   STATUS    RESTARTS   AGE   IP  
elasticsearch-0   1/1     Running   0          15m   10.244.0.126 
elasticsearch-1   1/1     Running   0          15m   10.244.0.127
elasticsearch-2   1/1     Running   0          15m   10.244.0.128
# 然后通过curl命令访问一下服务，验证es是否部署成功
$ kubectl -n logging get svc  
es-svc            ClusterIP   10.104.226.175   <none>        9200/TCP   2s
es-svc-headless   ClusterIP   None             <none>        9300/TCP   32m 
$ curl 10.104.226.175:9200
{
  "name" : "elasticsearch-2",
  "cluster_name" : "luffy-elasticsearch",
  "cluster_uuid" : "7FDIACx9T-2ajYcB5qp4hQ",
  "version" : {
    "number" : "7.4.2",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "2f90bbf7b93631e52bafb59b3b049cb44ec25e96",
    "build_date" : "2019-10-28T20:40:44.881551Z",
    "build_snapshot" : false,
    "lucene_version" : "8.2.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
```

##### 部署kibana

###### 部署分析

1. kibana需要暴露web页面给前端使用，因此使用ingress配置域名来实现对kibana的访问

2. kibana为无状态应用，直接使用Deployment来启动

3. kibana需要访问es，直接利用k8s服务发现访问此地址即可，http://es-svc:9200

###### 部署并验证

 `efk/kibana.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  selector:
    matchLabels:
      app: "kibana"
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: kibana:7.4.2
        resources:
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        env:
          - name: ELASTICSEARCH_HOSTS
            value: http://es-svc:9200
          - name: SERVER_NAME
            value: kibana-logging
          - name: SERVER_REWRITEBASEPATH
            value: "false"
        ports:
        - containerPort: 5601
        volumeMounts:
          - name: config
            mountPath: /usr/share/kibana/config/
      volumes:
        - name: config
          configMap:
            name: kibana-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: kibana-config
  namespace: logging
data:
  kibana.yml: |-
    elasticsearch.requestTimeout: 90000
    server.host: "0"
    xpack.monitoring.ui.container.elasticsearch.enabled: true
---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  ports:
  - port: 5601
    protocol: TCP
    targetPort: 5601
  type: ClusterIP
  selector:
    app: kibana
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kibana
  namespace: logging
spec:
  rules:
  - host: kibana.luffy.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: kibana
            port:
              number: 5601
```

```bash
$ kubectl apply -f kibana.yaml  
deployment.apps/kibana created
service/kibana created  
ingress/kibana created

## 配置域名解析 kibana.luffy.com，并访问服务进行验证，若可以访问，说明连接es成功


# GET /_cat/health?v
# GET /_cat/indices
```



##### Fluentd服务部署

###### 部署分析

1. fluentd为日志采集服务，kubernetes集群的每个业务节点都有日志产生，因此需要使用daemonset的模式进行部署
2. 为进一步控制资源，会为daemonset指定一个选择标签，fluentd=true来做进一步过滤，只有带有此标签的节点才会部署fluentd
3. 日志采集，需要采集哪些目录下的日志，采集后发送到es端，因此需要配置的内容比较多，我们选择使用configmap的方式把配置文件整个挂载出来

###### 部署服务

`efk/fluentd-es-config-main.yaml`

```yaml
apiVersion: v1
data:
  fluent.conf: |-
    # This is the root config file, which only includes components of the actual configuration
    #
    #  Do not collect fluentd's own logs to avoid infinite loops.
    <match fluent.**>
    @type null
    </match>

    @include /fluentd/etc/config.d/*.conf
kind: ConfigMap
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
  name: fluentd-es-config-main
  namespace: logging

```

配置文件，fluentd-config.yaml，注意点：

1. 数据源source的配置，k8s会默认把容器的标准和错误输出日志重定向到宿主机中
2. 默认集成了 [kubernetes_metadata_filter](https://github.com/fabric8io/fluent-plugin-kubernetes_metadata_filter) 插件，来解析日志格式，得到k8s相关的元数据，raw.kubernetes
3. match输出到es端的flush配置

`efk/fluentd-configmap.yaml`

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: fluentd-config
  namespace: logging
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
data:
  containers.input.conf: |-
    <source>
      @id fluentd-containers.log
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/es-containers.log.pos
      time_format %Y-%m-%dT%H:%M:%S.%NZ
      localtime
      tag raw.kubernetes.*
      format json
      read_from_head false
    </source>
    # Detect exceptions in the log output and forward them as one log entry.
    # https://github.com/GoogleCloudPlatform/fluent-plugin-detect-exceptions 
    <match raw.kubernetes.**>
      @id raw.kubernetes
      @type detect_exceptions
      remove_tag_prefix raw
      message log
      stream stream
      multiline_flush_interval 5
      max_bytes 500000
      max_lines 1000
    </match>
        # Concatenate multi-line logs
    <filter **>
      @id filter_concat
      @type concat
      key message
      multiline_end_regexp /\n$/
      separator ""
    </filter>
  output.conf: |-
    # Enriches records with Kubernetes metadata
    <filter kubernetes.**>
      @type kubernetes_metadata
    </filter>
    <match **>
      @id elasticsearch
      @type elasticsearch
      @log_level info
      include_tag_key true
      hosts elasticsearch-0.es-svc-headless:9200,elasticsearch-1.es-svc-headless:9200,elasticsearch-2.es-svc-headless:9200
      #port 9200
      logstash_format true
      #index_name kubernetes-%Y.%m.%d
      request_timeout    30s
      <buffer>
        @type file
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff
        flush_thread_count 2
        flush_interval 5s
        retry_forever
        retry_max_interval 30
        chunk_limit_size 2M
        queue_limit_length 8
        overflow_action block
      </buffer>
    </match>
```

daemonset定义文件，fluentd.yaml，注意点：

1. 需要配置rbac规则，因为需要访问k8s api去根据日志查询元数据
2. 需要将/var/log/containers/目录挂载到容器中
3. 需要将fluentd的configmap中的配置文件挂载到容器内
4. 想要部署fluentd的节点，需要添加fluentd=true的标签

`efk/fluentd.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd-es
  namespace: logging
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
rules:
- apiGroups:
  - ""
  resources:
  - "namespaces"
  - "pods"
  verbs:
  - "get"
  - "watch"
  - "list"
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd-es
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
subjects:
- kind: ServiceAccount
  name: fluentd-es
  namespace: logging
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: fluentd-es
  apiGroup: ""
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    k8s-app: fluentd-es
  name: fluentd-es
  namespace: logging
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-es
  template:
    metadata:
      labels:
        k8s-app: fluentd-es
    spec:
      containers:
      - image: quay.io/fluentd_elasticsearch/fluentd:v3.1.0
        imagePullPolicy: IfNotPresent
        name: fluentd-es
        resources:
          limits:
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - mountPath: /var/log
          name: varlog
        - mountPath: /var/lib/docker/containers
          name: varlibdockercontainers
          readOnly: true
        - mountPath: /etc/fluent/config.d
          name: config-volume
      nodeSelector:
        fluentd: "true"
      securityContext: {}
      serviceAccount: fluentd-es
      serviceAccountName: fluentd-es
      volumes:
      - hostPath:
          path: /var/log
        name: varlog
      - hostPath:
          path: /var/lib/docker/containers
        name: varlibdockercontainers
      - configMap:
          defaultMode: 420
          name: fluentd-config
        name: config-volume
```

```bash
## 给slave1打上标签，进行部署fluentd日志采集服务
$ kubectl label node k8s-slave1 fluentd=true  
$ kubectl label node k8s-slave2 fluentd=true

# 创建服务
$ kubectl apply -f fluentd-es-config-main.yaml  
configmap/fluentd-es-config-main created  
$ kubectl apply -f fluentd-configmap.yaml  
configmap/fluentd-config created  
$ kubectl apply -f fluentd.yaml  
serviceaccount/fluentd-es created  
clusterrole.rbac.authorization.k8s.io/fluentd-es created  
clusterrolebinding.rbac.authorization.k8s.io/fluentd-es created  
daemonset.extensions/fluentd-es created 

## 然后查看一下pod是否已经在k8s-slave1
$ kubectl -n logging get po -o wide
NAME                      READY   STATUS    RESTARTS   AGE  
elasticsearch-logging-0   1/1     Running   0          123m  
fluentd-es-246pl   		  1/1     Running   0          2m2s  
kibana-944c57766-ftlcw    1/1     Running   0          50m
```

