##### 实例介绍

创建bookinfo实例：

```bash
$ kubectl create namespace bookinfo
$ kubectl -n bookinfo create -f samples/bookinfo/platform/kube/bookinfo.yaml 
$ kubectl -n bookinfo get po 
NAME                                  READY   STATUS    RESTARTS   AGE
details-v1-5974b67c8-wclnd            1/1     Running   0          34s
productpage-v1-64794f5db4-jsdbg       1/1     Running   0          33s
ratings-v1-c6cdf8d98-jrfrn            1/1     Running   0          33s
reviews-v1-7f6558b974-kq6kj           1/1     Running   0          33s
reviews-v2-6cb6ccd848-qdg2k           1/1     Running   0          34s
reviews-v3-cc56b578-kppcx             1/1     Running   0          34s
```

该应用由四个单独的微服务构成。 这个应用模仿在线书店的一个分类，显示一本书的信息。 页面上会显示一本书的描述，书籍的细节（ISBN、页数等），以及关于这本书的一些评论。

Bookinfo 应用分为四个单独的微服务：

- `productpage`. 这个微服务会调用 `details` 和 `reviews` 两个微服务，用来生成页面。
- `details`. 这个微服务中包含了书籍的信息。
- `reviews`. 这个微服务中包含了书籍相关的评论。它还会调用 `ratings` 微服务。
- `ratings`. 这个微服务中包含了由书籍评价组成的评级信息。

`reviews` 微服务有 3 个版本：

- v1 版本不会调用 `ratings` 服务。
- v2 版本会调用 `ratings` 服务，并使用 1 到 5 个黑色星形图标来显示评分信息。
- v3 版本会调用 `ratings` 服务，并使用 1 到 5 个红色星形图标来显示评分信息。

![](images\noistio.svg)

Bookinfo 是一个异构应用，几个微服务是由不同的语言编写的。这些服务对 Istio 并无依赖，但是构成了一个有代表性的服务网格的例子：它由多个服务、多个语言构成，并且 `reviews` 服务具有多个版本。 

使用ingress访问productpage服务：

`ingress-productpage.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: productpage
  namespace: bookinfo
spec:
  rules:
  - host: bookinfo.luffy.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: productpage
            port:
              number: 9080
```



如何实现更细粒度的流量管控？



##### 注入sidecar容器

###### 如何注入sidecar容器

1. 使用`istioctl kube-inject`

   ```bash
   $ kubectl -n bookinfo apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)
   ```

2. 为命名空间打label

   ```bash
   # 给命名空间打标签，这样部署在该命名空间的服务会自动注入sidecar容器
   $ kubectl label namespace dafault istio-injection=enabled
   ```

###### 注入bookinfo

```bash
$ kubectl -n bookinfo apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml) 
```

![](images\withistio.svg)
