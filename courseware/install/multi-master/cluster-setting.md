### 设置master节点是否可调度（可选）

操作节点：`k8s-master`

默认部署成功后，master节点无法调度业务pod，如需设置master节点也可以参与pod的调度，需执行：

``` python
$ kubectl taint node k8s-master node-role.kubernetes.io/master:NoSchedule-
```

> 课程后期会部署系统组件到master节点，因此，此处建议设置k8s-master节点为可调度

### 设置kubectl自动补全

操作节点：`k8s-master`

```bash
$ yum install bash-completion -y
$ source /usr/share/bash-completion/bash_completion
$ source <(kubectl completion bash)
$ echo "source <(kubectl completion bash)" >> ~/.bashrc
```

