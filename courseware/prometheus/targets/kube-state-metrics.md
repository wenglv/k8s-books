###### kube-state-metrics监控

已经有了cadvisor，容器运行的指标已经可以获取到，但是下面这种情况却无能为力：

- 我调度了多少个replicas？现在可用的有几个？
- 多少个Pod是running/stopped/terminated状态？
- Pod重启了多少次？

而这些则是kube-state-metrics提供的内容，它基于client-go开发，轮询Kubernetes API，并将Kubernetes的结构化信息转换为metrics。因此，需要借助于`kube-state-metrics`来实现。

指标类别包括：

- CronJob Metrics
- DaemonSet Metrics
- Deployment Metrics
- Job Metrics
- LimitRange Metrics
- Node Metrics
- PersistentVolume Metrics
- PersistentVolumeClaim Metrics
- Pod Metrics
  - kube_pod_info
  - kube_pod_owner
  - kube_pod_status_phase
  - kube_pod_status_ready
  - kube_pod_status_scheduled
  - kube_pod_container_status_waiting
  - kube_pod_container_status_terminated_reason
  - ...
- Pod Disruption Budget Metrics
- ReplicaSet Metrics
- ReplicationController Metrics
- ResourceQuota Metrics
- Service Metrics
- StatefulSet Metrics
- Namespace Metrics
- Horizontal Pod Autoscaler Metrics
- Endpoint Metrics
- Secret Metrics
- ConfigMap Metrics

部署： https://github.com/kubernetes/kube-state-metrics#kubernetes-deployment 

```bash
$ wget https://github.com/kubernetes/kube-state-metrics/archive/v2.1.0.tar.gz

$ tar zxf kube-state-metrics-2.1.0.tar.gz
$  cp -r kube-state-metrics-2.1.0/examples/standard/ .

$ ll standard/
total 20
-rw-r--r-- 1 root root  376 Jun 24 20:49 cluster-role-binding.yaml
-rw-r--r-- 1 root root 1623 Jun 24 20:49 cluster-role.yaml
-rw-r--r-- 1 root root 1134 Jun 24 20:49 deployment.yaml
-rw-r--r-- 1 root root  192 Jun 24 20:49 service-account.yaml
-rw-r--r-- 1 root root  405 Jun 24 20:49 service.yaml

# 替换namespace为monitor
$ sed -i 's/namespace: kube-system/namespace: monitor/g' standard/*

# 替换镜像地址为image: bitnami/kube-state-metrics:2.1.0
$ sed -i 's#k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.1.0#bitnami/kube-state-metrics:2.1.0#g' standard/deployment.yaml

$ kubectl apply -f standard/
clusterrolebinding.rbac.authorization.k8s.io/kube-state-metrics created
clusterrole.rbac.authorization.k8s.io/kube-state-metrics created
deployment.apps/kube-state-metrics created
serviceaccount/kube-state-metrics created
service/kube-state-metrics created
```



如何添加到Prometheus监控target中？

```bash
$ cat standard/service.yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
  labels:
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.1.0
  name: kube-state-metrics
  namespace: monitor
spec:
  clusterIP: None
  ports:
  - name: http-metrics
    port: 8080
    targetPort: http-metrics
  - name: telemetry
    port: 8081
    targetPort: telemetry
  selector:
    app.kubernetes.io/name: kube-state-metrics
    
$ kubectl apply -f standard/service.yaml
```



查看target列表，观察是否存在kube-state-metrics的target。

kube_pod_container_status_running

kube_deployment_status_replicas

kube_deployment_status_replicas_unavailable