操作节点： 在master节点（`k8s-master`）执行

``` python
$ kubectl get nodes  #观察集群节点是否全部Ready
```

创建测试nginx服务

``` python
$ kubectl run  test-nginx --image=nginx:alpine
```

查看pod是否创建成功，并访问pod ip测试是否可用

``` bash
$ kubectl get po -o wide
NAME                          READY   STATUS    RESTARTS   AGE   IP           NODE         NOMINATED NODE   READINESS GATES
test-nginx-5bd8859b98-5nnnw   1/1     Running   0          9s    10.244.1.2   k8s-slave1   <none>           <none>
$ curl 10.244.1.2
...
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

