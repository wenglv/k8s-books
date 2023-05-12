#### Kubernetes认证与授权 

###### APIServer安全控制

![](images/k8s-apiserver-access-control-overview.svg)

- Authentication：身份认证

  1. 这个环节它面对的输入是整个`http request`，负责对来自client的请求进行身份校验，支持的方法包括:

     - `basic auth`

     - client证书验证（https双向验证）

     - `jwt token`(用于serviceaccount)

  2. APIServer启动时，可以指定一种Authentication方法，也可以指定多种方法。如果指定了多种方法，那么APIServer将会逐个使用这些方法对客户端请求进行验证， 只要请求数据通过其中一种方法的验证，APIServer就会认为Authentication成功；

  3. 使用kubeadm引导启动的k8s集群，apiserver的初始配置中，默认支持`client证书`验证和`serviceaccount`两种身份验证方式。 证书认证通过设置`--client-ca-file`根证书以及`--tls-cert-file`和`--tls-private-key-file`来开启。

  4. 在这个环节，apiserver会通过client证书或 `http header`中的字段(比如serviceaccount的`jwt token`)来识别出请求的`用户身份`，包括”user”、”group”等，这些信息将在后面的`authorization`环节用到。

- Authorization：鉴权，你可以访问哪些资源

  1. 这个环节面对的输入是`http request context`中的各种属性，包括：`user`、`group`、`request path`（比如：`/api/v1`、`/healthz`、`/version`等）、 `request verb`(比如：`get`、`list`、`create`等)。

  2. APIServer会将这些属性值与事先配置好的访问策略(`access policy`）相比较。APIServer支持多种`authorization mode`，包括`Node、RBAC、Webhook`等。

  3. APIServer启动时，可以指定一种`authorization mode`，也可以指定多种`authorization mode`，如果是后者，只要Request通过了其中一种mode的授权， 那么该环节的最终结果就是授权成功。在较新版本kubeadm引导启动的k8s集群的apiserver初始配置中，`authorization-mode`的默认配置是`”Node,RBAC”`。

- Admission Control：[准入控制](http://docs.kubernetes.org.cn/144.html)，一个控制链(层层关卡)，用于拦截请求的一种方式。偏集群安全控制、管理方面。

  - 为什么需要？

    认证与授权获取 http 请求 header 以及证书，无法通过body内容做校验。

    Admission 运行在 API Server 的增删改查 handler 中，可以自然地操作 API resource 

  - 举个栗子

    - 以NamespaceLifecycle为例， 该插件确保处于Termination状态的Namespace不再接收新的对象创建请求，并拒绝请求不存在的Namespace。该插件还可以防止删除系统保留的Namespace:default，kube-system，kube-public

    - LimitRanger，若集群的命名空间设置了LimitRange对象，若Pod声明时未设置资源值，则按照LimitRange的定义来未Pod添加默认值

      ```yaml
      apiVersion: v1
      kind: LimitRange
      metadata:
        name: mem-limit-range
        namespace: demo
      spec:
        limits:
        - default:
            memory: 512Mi
          defaultRequest:
            memory: 256Mi
          type: Container
      ---
      apiVersion: v1
      kind: Pod
      metadata:
        name: default-mem-demo
        namespace: demo
      spec:
        containers:
        - name: default-mem-demo
          image: nginx:alpine
      
      ```

    ```bash
    $ kubectl apply -f .
    ```

    

  - NodeRestriction， 此插件限制kubelet修改Node和Pod对象，这样的kubelets只允许修改绑定到Node的Pod API对象，以后版本可能会增加额外的限制 。开启Node授权策略后，默认会打开该项

- 怎么用？

  APIServer启动时通过 `--enable-admission-plugins --disable-admission-plugins` 指定需要打开或者关闭的 Admission Controller 

- 场景

  - 自动注入sidecar容器或者initContainer容器
  - webhook admission，实现业务自定义的控制需求



###### kubectl的认证授权

kubectl的日志调试级别：

| 信息 | 描述                                                         |
| :--- | :----------------------------------------------------------- |
| v=0  | 通常，这对操作者来说总是可见的。                             |
| v=1  | 当您不想要很详细的输出时，这个是一个合理的默认日志级别。     |
| v=2  | 有关服务和重要日志消息的有用稳定状态信息，这些信息可能与系统中的重大更改相关。这是大多数系统推荐的默认日志级别。 |
| v=3  | 关于更改的扩展信息。                                         |
| v=4  | 调试级别信息。                                               |
| v=6  | 显示请求资源。                                               |
| v=7  | 显示 HTTP 请求头。                                           |
| v=8  | 显示 HTTP 请求内容。                                         |
| v=9  | 显示 HTTP 请求内容，并且不截断内容。                         |

```bash
$ kubectl get nodes -v=7
I0329 20:20:08.633065    3979 loader.go:359] Config loaded from file /root/.kube/config
I0329 20:20:08.633797    3979 round_trippers.go:416] GET https://172.21.51.143:6443/api/v1/nodes?limit=500


```

`kubeadm init`启动完master节点后，会默认输出类似下面的提示内容：

```bash
... ...
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
... ...


```

这些信息是在告知我们如何配置`kubeconfig`文件。按照上述命令配置后，master节点上的`kubectl`就可以直接使用`$HOME/.kube/config`的信息访问`k8s cluster`了。 并且，通过这种配置方式，`kubectl`也拥有了整个集群的管理员(root)权限。

很多K8s初学者在这里都会有疑问：

- 当`kubectl`使用这种`kubeconfig`方式访问集群时，`Kubernetes`的`kube-apiserver`是如何对来自`kubectl`的访问进行身份验证(`authentication`)和授权(`authorization`)的呢？
- 为什么来自`kubectl`的请求拥有最高的管理员权限呢？ 

查看`/root/.kube/config`文件：

前面提到过apiserver的authentication支持通过`tls client certificate、basic auth、token`等方式对客户端发起的请求进行身份校验， 从kubeconfig信息来看，kubectl显然在请求中使用了`tls client certificate`的方式，即客户端的证书。 

证书base64解码：

```bash
$ echo xxxxxxxxxxxxxx |base64 -d > kubectl.crt

```

说明在认证阶段，`apiserver`会首先使用`--client-ca-file`配置的CA证书去验证kubectl提供的证书的有效性,基本的方式 ：

```bash
$  openssl verify -CAfile /etc/kubernetes/pki/ca.crt kubectl.crt
kubectl.crt: OK

```

除了认证身份，还会取出必要的信息供授权阶段使用，文本形式查看证书内容：

```bash
$ openssl x509 -in kubectl.crt -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4736260165981664452 (0x41ba9386f52b74c4)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes
        Validity
            Not Before: Feb 10 07:33:39 2020 GMT
            Not After : Feb  9 07:33:40 2021 GMT
        Subject: O=system:masters, CN=kubernetes-admin
        ...



```

认证通过后，提取出签发证书时指定的CN(Common Name),`kubernetes-admin`，作为请求的用户名 (User Name), 从证书中提取O(Organization)字段作为请求用户所属的组 (Group)，`group = system:masters`，然后传递给后面的授权模块。

kubeadm在init初始引导集群启动过程中，创建了许多默认的RBAC规则， 在k8s有关RBAC的官方文档中，我们看到下面一些`default clusterrole`列表: 

![](images/kubeadm-default-clusterrole-list.png)

其中第一个cluster-admin这个cluster role binding绑定了system:masters group，这和authentication环节传递过来的身份信息不谋而合。 沿着system:masters group对应的cluster-admin clusterrolebinding“追查”下去，真相就会浮出水面。

我们查看一下这一binding：

```yaml
$ kubectl describe clusterrolebinding cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  cluster-admin
Subjects:
  Kind   Name            Namespace
  ----   ----            ---------
  Group  system:masters



```

我们看到在kube-system名字空间中，一个名为cluster-admin的clusterrolebinding将cluster-admin cluster role与system:masters Group绑定到了一起， 赋予了所有归属于system:masters Group中用户cluster-admin角色所拥有的权限。

我们再来查看一下cluster-admin这个role的具体权限信息：

```bash
$ kubectl describe clusterrole cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]



```

非资源类，如查看集群健康状态。

![](images/how-kubectl-be-authorized.png)



###### RBAC

Role-Based Access Control，基于角色的访问控制， apiserver启动参数添加--authorization-mode=RBAC 来启用RBAC认证模式，kubeadm安装的集群默认已开启。[官方介绍](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

查看开启：

```bash
# master节点查看apiserver进程
$ ps aux |grep apiserver


```

RBAC模式引入了4个资源类型：

- Role，角色

  一个Role只能授权访问单个namespace 

  ```yaml
  ## 示例定义一个名为pod-reader的角色，该角色具有读取demo这个命名空间下的pods的权限
  kind: Role
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    namespace: demo
    name: pod-reader
  rules:
  - apiGroups: [""] # "" indicates the core API group
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
    
  ## apiGroups: "","apps", "autoscaling", "batch", kubectl api-versions
  ## resources: "services", "pods","deployments"... kubectl api-resources
  ## verbs: "get", "list", "watch", "create", "update", "patch", "delete", "exec"
  
  ## https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/
  
  ```

- ClusterRole

  一个ClusterRole能够授予和Role一样的权限，但是它是集群范围内的。 

  ```yaml
  ## 定义一个集群角色，名为secret-reader，该角色可以读取所有的namespace中的secret资源
  kind: ClusterRole
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    # "namespace" omitted since ClusterRoles are not namespaced
    name: secret-reader
  rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "watch", "list"]
  
  # User,Group,ServiceAccount
  ```

- Rolebinding

  将role中定义的权限分配给用户和用户组。RoleBinding包含主题（users,groups,或service accounts）和授予角色的引用。对于namespace内的授权使用RoleBinding，集群范围内使用ClusterRoleBinding。

  ```yaml
  ## 定义一个角色绑定，将pod-reader这个role的权限授予给jane这个User，使得jane可以在读取default这个命名空间下的所有的pod数据
  kind: RoleBinding
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: read-pods
    namespace: demo
  subjects:
  - kind: User   #这里可以是User,Group,ServiceAccount
    name: jane 
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: ClusterRole #这里可以是Role或者ClusterRole,若是ClusterRole，则权限也仅限于rolebinding的内部
    name: secret-reader # match the name of the Role or ClusterRole you wish to bind to
    apiGroup: rbac.authorization.k8s.io
  
  
  ```

  *注意：rolebinding既可以绑定role，也可以绑定clusterrole，当绑定clusterrole的时候，subject的权限也会被限定于rolebinding定义的namespace内部，若想跨namespace，需要使用clusterrolebinding*

  ```yaml
  ## 定义一个角色绑定，将dave这个用户和secret-reader这个集群角色绑定，虽然secret-reader是集群角色，但是因为是使用rolebinding绑定的，因此dave的权限也会被限制在development这个命名空间内
  apiVersion: rbac.authorization.k8s.io/v1
  # This role binding allows "dave" to read secrets in the "development" namespace.
  # You need to already have a ClusterRole named "secret-reader".
  kind: RoleBinding
  metadata:
    name: read-secrets
    #
    # The namespace of the RoleBinding determines where the permissions are granted.
    # This only grants permissions within the "development" namespace.
    namespace: development
  subjects:
  - kind: User
    name: dave # Name is case sensitive
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: dave # Name is case sensitive
    namespace: luffy
  roleRef:
    kind: ClusterRole
    name: secret-reader
    apiGroup: rbac.authorization.k8s.io
  
  
  ```

  考虑一个场景：  如果集群中有多个namespace分配给不同的管理员，每个namespace的权限是一样的，就可以只定义一个clusterrole，然后通过rolebinding将不同的namespace绑定到管理员身上，否则就需要每个namespace定义一个Role，然后做一次rolebinding。

- ClusterRolebingding

  允许跨namespace进行授权

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  # This cluster role binding allows anyone in the "manager" group to read secrets in any namespace.
  kind: ClusterRoleBinding
  metadata:
    name: read-secrets-global
  subjects:
  - kind: Group
    name: manager # Name is case sensitive
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: ClusterRole
    name: secret-reader
    apiGroup: rbac.authorization.k8s.io
  
  
  ```

![](images/rbac-2.jpg)

###### kubelet的认证授权

查看kubelet进程

```bash
$ systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Sun 2020-07-05 19:33:36 EDT; 1 day 12h ago
     Docs: https://kubernetes.io/docs/
 Main PID: 10622 (kubelet)
    Tasks: 24
   Memory: 60.5M
   CGroup: /system.slice/kubelet.service
           └─851 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf



```

查看`/etc/kubernetes/kubelet.conf`，解析证书：

```bash
$ echo xxxxx |base64 -d >kubelet.crt
$ openssl x509 -in kubelet.crt -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 9059794385454520113 (0x7dbadafe23185731)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes
        Validity
            Not Before: Feb 10 07:33:39 2020 GMT
            Not After : Feb  9 07:33:40 2021 GMT
        Subject: O=system:nodes, CN=system:node:master-1



```

得到我们期望的内容：

```bash
Subject: O=system:nodes, CN=system:node:k8s-master

```

我们知道，k8s会把O作为Group来进行请求，因此如果有权限绑定给这个组，肯定在clusterrolebinding的定义中可以找得到。因此尝试去找一下绑定了system:nodes组的clusterrolebinding

```bash
$ kubectl get clusterrolebinding -oyaml|grep -n10 system:nodes
178-    resourceVersion: "225"
179-    selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/kubeadm%3Anode-autoapprove-certificate-rotation
180-    uid: b4303542-d383-4b62-a1e9-02f2cefa2c20
181-  roleRef:
182-    apiGroup: rbac.authorization.k8s.io
183-    kind: ClusterRole
184-    name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
185-  subjects:
186-  - apiGroup: rbac.authorization.k8s.io
187-    kind: Group
188:    name: system:nodes
189-- apiVersion: rbac.authorization.k8s.io/v1
190-  kind: ClusterRoleBinding
191-  metadata:
192-    creationTimestamp: "2021-06-06T02:39:46Z"
193-    managedFields:
194-    - apiVersion: rbac.authorization.k8s.io/v1
195-      fieldsType: FieldsV1
196-      fieldsV1:
197-        f:roleRef:
198-          f:apiGroup: {}

$ kubectl describe clusterrole system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
Name:         system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources                                                      Non-Resource URLs  Resource Names  Verbs
  ---------                                                      -----------------  --------------  -----
  certificatesigningrequests.certificates.k8s.io/selfnodeclient  []                 []              [create]

```

 结局有点意外，除了`system:certificates.k8s.io:certificatesigningrequests:selfnodeclient`外，没有找到system相关的rolebindings，显然和我们的理解不一样。 尝试去找[资料](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#core-component-roles)，发现了这么一段 :

| Default ClusterRole            | Default ClusterRoleBinding          | Description                                                  |
| :----------------------------- | :---------------------------------- | :----------------------------------------------------------- |
| system:kube-scheduler          | system:kube-scheduler user          | Allows access to the resources required by the [scheduler](https://kubernetes.io/docs/reference/generated/kube-scheduler/)component. |
| system:volume-scheduler        | system:kube-scheduler user          | Allows access to the volume resources required by the kube-scheduler component. |
| system:kube-controller-manager | system:kube-controller-manager user | Allows access to the resources required by the [controller manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/) component. The permissions required by individual controllers are detailed in the [controller roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#controller-roles). |
| system:node                    | None                                | Allows access to resources required by the kubelet, **including read access to all secrets, and write access to all pod status objects**. You should use the [Node authorizer](https://kubernetes.io/docs/reference/access-authn-authz/node/) and [NodeRestriction admission plugin](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction) instead of the `system:node` role, and allow granting API access to kubelets based on the Pods scheduled to run on them. The `system:node` role only exists for compatibility with Kubernetes clusters upgraded from versions prior to v1.8. |
| system:node-proxier            | system:kube-proxy user              | Allows access to the resources required by the [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)component. |

大致意思是说：之前会定义system:node这个角色，目的是为了kubelet可以访问到必要的资源，包括所有secret的读权限及更新pod状态的写权限。如果1.8版本后，是建议使用 [Node authorizer](https://kubernetes.io/docs/reference/access-authn-authz/node/) and [NodeRestriction admission plugin](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction) 来代替这个角色的。

我们目前使用1.19，查看一下授权策略：

```bash
$ ps axu|grep apiserver
kube-apiserver --authorization-mode=Node,RBAC  --enable-admission-plugins=NodeRestriction


```

查看一下官网对Node authorizer的介绍：

*Node authorization is a special-purpose authorization mode that specifically authorizes API requests made by kubelets.*

*In future releases, the node authorizer may add or remove permissions to ensure kubelets have the minimal set of permissions required to operate correctly.*

*In order to be authorized by the Node authorizer, kubelets must use a credential that identifies them as being in the `system:nodes` group, with a username of `system:node:<nodeName>`*



###### Service Account及K8S Api调用

前面说，认证可以通过证书，也可以通过使用ServiceAccount（服务账户）的方式来做认证。大多数时候，我们在基于k8s做二次开发时都是选择通过ServiceAccount + RBAC 的方式。我们之前访问dashboard的时候，是如何做的？

```yaml
## 新建一个名为admin的serviceaccount，并且把名为cluster-admin的这个集群角色的权限授予新建的
#serviceaccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kubernetes-dashboard
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kubernetes-dashboard


```

我们查看一下：

```bash
$ kubectl -n kubernetes-dashboard get sa admin -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2020-04-01T11:59:21Z"
  name: admin
  namespace: kubernetes-dashboard
  resourceVersion: "1988878"
  selfLink: /api/v1/namespaces/kubernetes-dashboard/serviceaccounts/admin
  uid: 639ecc3e-74d9-11ea-a59b-000c29dfd73f
secrets:
- name: admin-token-lfsrf


```

注意到serviceaccount上默认绑定了一个名为admin-token-lfsrf的secret，我们查看一下secret

```bash
$ kubectl -n kubernetes-dashboard describe secret admin-token-lfsrf
Name:         admin-token-lfsrf
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin
              kubernetes.io/service-account.uid: 639ecc3e-74d9-11ea-a59b-000c29dfd73f

Type:  kubernetes.io/service-account-token
Data
====
ca.crt:     1025 bytes
namespace:  4 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZW1vIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImFkbWluLXRva2VuLWxmc3JmIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNjM5ZWNjM2UtNzRkOS0xMWVhLWE1OWItMDAwYzI5ZGZkNzNmIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlbW86YWRtaW4ifQ.ffGCU4L5LxTsMx3NcNixpjT6nLBi-pmstb4I-W61nLOzNaMmYSEIwAaugKMzNR-2VwM14WbuG04dOeO67niJeP6n8-ALkl-vineoYCsUjrzJ09qpM3TNUPatHFqyjcqJ87h4VKZEqk2qCCmLxB6AGbEHpVFkoge40vHs56cIymFGZLe53JZkhu3pwYuS4jpXytV30Ad-HwmQDUu_Xqcifni6tDYPCfKz2CZlcOfwqHeGIHJjDGVBKqhEeo8PhStoofBU6Y4OjObP7HGuTY-Foo4QindNnpp0QU6vSb7kiOiQ4twpayybH8PTf73dtdFt46UF6mGjskWgevgolvmO8A


```



![](images\rbac.jpg)

只允许访问luffy命名空间的pod资源：

```bash
$ cat luffy-admin-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: luffy-pods-admin
  namespace: luffy

---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: luffy
  name: pods-reader-writer
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["*"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pods-reader-writer
  namespace: luffy
subjects:
- kind: ServiceAccount   #这里可以是User,Group,ServiceAccount
  name: luffy-pods-admin
  namespace: luffy
roleRef:
  kind: Role #这里可以是Role或者ClusterRole,若是ClusterRole，则权限也仅限于rolebinding的内部
  name: pods-reader-writer
  apiGroup: rbac.authorization.k8s.io
```



演示权限：

```bash
$ kubectl -n luffy describe secrets luffy-pods-admin-token-prr25
...
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6InBtQUZfRl8ycC03TTBYaUUwTnJVZGpvQWU0cXZ5M2FFbjR2ZjkzZVcxOE0ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJsdWZmeSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJsdWZmeS1hZG1pbi10b2tlbi1wcnIyNSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJsdWZmeS1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImFhZDA0MTU3LTliNzMtNDJhZC1hMGU4LWVmOTZlZDU3Yzg1ZiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpsdWZmeTpsdWZmeS1hZG1pbiJ9.YWckylE5wlKITKrVltXY4VPKvZP9ar5quIT5zq9N-0_FnDkLIBX7xOyFvZA5Wef0wSFSZe3e9FwrO1UbPsmK7cZn74bhH8cNdoH_YVbIVT3-6tIOlCA_Bc8YypGE1gl-ZvLOIPV7WnRQsWpWtZtqfKBSkwLAHgWoxcx_d1bOcyTOdPmsW224xcBxjYwi6iRUtjTJST0LzOcAOCPDZq6-lqYUwnxLO_afxwg71BGX4etE48Iny8TxSEIs1VJRahoabC7hVOs17ujEm5loTDSpfuhae51qSDg8xeYwRHdM42aLUmc-wOvBWauHa5EHbH9rWPAnpaGIwF8QvnLszqp4QQ
...
$ curl -k  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6InBtQUZfRl8ycC03TTBYaUUwTnJVZGpvQWU0cXZ5M2FFbjR2ZjkzZVcxOE0ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJsdWZmeSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJsdWZmeS1hZG1pbi10b2tlbi1wcnIyNSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJsdWZmeS1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImFhZDA0MTU3LTliNzMtNDJhZC1hMGU4LWVmOTZlZDU3Yzg1ZiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpsdWZmeTpsdWZmeS1hZG1pbiJ9.YWckylE5wlKITKrVltXY4VPKvZP9ar5quIT5zq9N-0_FnDkLIBX7xOyFvZA5Wef0wSFSZe3e9FwrO1UbPsmK7cZn74bhH8cNdoH_YVbIVT3-6tIOlCA_Bc8YypGE1gl-ZvLOIPV7WnRQsWpWtZtqfKBSkwLAHgWoxcx_d1bOcyTOdPmsW224xcBxjYwi6iRUtjTJST0LzOcAOCPDZq6-lqYUwnxLO_afxwg71BGX4etE48Iny8TxSEIs1VJRahoabC7hVOs17ujEm5loTDSpfuhae51qSDg8xeYwRHdM42aLUmc-wOvBWauHa5EHbH9rWPAnpaGIwF8QvnLszqp4QQ" https://172.21.51.143:6443/api/v1/namespaces/luffy/pods?limit=500

# https://172.21.51.143:6443/api/v1/nodes
```

###### 认证、鉴权图鉴

![](images\AUTH.jpg)

###### 创建用户认证授权的kubeconfig文件

签发证书对：

```bash
# 生成私钥
$ openssl genrsa -out luffy.key 2048

# 生成证书请求文件
$ openssl req -new -key luffy.key -out luffy.csr -subj "/O=admin:luffy/CN=luffy-admin"

# 证书拓展属性
$ cat extfile.conf
[ v3_ca ]
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth

# 生成luffy.crt证书
$ openssl x509 -req -in luffy.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -sha256 -out luffy.crt -extensions v3_ca -extfile extfile.conf -days 3650
```

配置kubeconfig文件：

```bash
# 创建kubeconfig文件，指定集群名称和地址
$ kubectl config set-cluster luffy-cluster --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true --server=https://172.21.51.143:6443 --kubeconfig=luffy.kubeconfig

# 为kubeconfig文件添加认证信息
$ kubectl config set-credentials luffy-admin --client-certificate=luffy.crt --client-key=luffy.key --embed-certs=true --kubeconfig=luffy.kubeconfig

# 为kubeconfig添加上下文配置
$ kubectl config set-context luffy-context --cluster=luffy-cluster --user=luffy-admin --kubeconfig=luffy.kubeconfig

# 设置默认的上下文
$ kubectl config use-context luffy-context --kubeconfig=luffy.kubeconfig
```



验证：

```bash
# 设置当前kubectl使用的config文件
$ export KUBECONFIG=luffy.kubeconfig

# 当前不具有任何权限，因为没有为用户或者组设置RBAC规则
$ kubectl get po
Error from server (Forbidden): pods is forbidden: User "luffy-admin" cannot list resource "pods" in API group "" in the namespace "default"

```



为luffy-admin用户添加luffy命名空间访问权限：

```bash
# 定义role，具有luffy命名空间的所有权限
$ cat luffy-admin-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: luffy
  name: luffy-admin
rules:
- apiGroups: ["*"] # "" 指定核心 API 组
  resources: ["*"]
  verbs: ["*"]
  
#定义rolebinding，为luffy用户绑定luffy-admin这个role，这样luffy用户就有操作luffy命名空间的所有权限
$ cat luffy-admin-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: luffy-admin
  namespace: luffy
subjects:
- kind: User
  name: luffy-admin # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: luffy-admin # 这里的名称必须与你想要绑定的 Role 或 ClusterRole 名称一致
  apiGroup: rbac.authorization.k8s.io
```



