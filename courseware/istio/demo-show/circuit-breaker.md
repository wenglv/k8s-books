##### 重试

在网络环境不稳定的情况下，会出现暂时的网络不可达现象，这时需要重试机制，通过多次尝试来获取正确的返回信息。 istio 可以通过简单的配置来实现重试功能，让开发人员无需关注重试部分的代码实现，专心实现业务代码。 

###### 实践

浏览器访问`http://www.bookinfo.com/httpbin/status/502` 

```bash
# 此时查看httpbin-v1的日志，显示一条状态码为502的日志
$ kubectl -n bookinfo logs -f httpbin-v1-5967569c54-sp874 -c istio-proxy
[2020-11-09T10:26:48.907Z] "GET /httpbin/status/502 HTTP/1.1" 502 - "-" "-" 0 0 5 4 "172.21.50.140,10.244.0.1" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.183 Safari/537.36" "767cf33b-b8cf-9804-8c48-df131393c8a5" "bookinfo.com" "127.0.0.1:80" inbound|8000|http|httpbin.bookinfo.svc.cluster.local 127.0.0.1:48376 10.244.0.90:80 10.244.0.1:0 outbound_.8000_.v1_.httpbin.bookinfo.svc.cluster.local default
```



我们为`httpbin`服务设置重试机制，这里设置如果服务在 2 秒内没有返回正确的返回值，就进行重试，重试的条件为返回码为`5xx`，重试 3 次。

```bash
$ kubectl -n bookinfo edit vs bookinfo
...
  - match:
    - uri:
        prefix: /httpbin
    mirror:
      host: httpbin
      subset: v2
    mirror_percent: 100
    name: httpbin-route
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx
    rewrite:
      uri: /
    route:
    - destination:
        host: httpbin
        subset: v1
...

# 再次查看httpbin-v1的日志，显示四条状态码为502的日志
```