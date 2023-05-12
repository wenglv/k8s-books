##### 故障注入与超时机制

在一个微服务架构的系统中，为了让系统达到较高的健壮性要求，通常需要对系统做定向错误测试。比如电商中的订单系统、支付系统等若出现故障那将是非常严重的生产事故，因此必须在系统设计前期就需要考虑多样性的异常故障并对每一种异常设计完善的恢复策略或优雅的回退策略，尽全力规避类似事故的发生，使得当系统发生故障时依然可以正常运作。而在这个过程中，服务故障模拟一直以来是一个非常繁杂的工作。 

istio提供了无侵入式的故障注入机制，让开发测试人员在不用调整服务程序的前提下，通过配置即可完成对服务的异常模拟。目前，包含两类：

-  **abort**：非必配项，配置一个 Abort 类型的对象。用来注入请求异常类故障。简单的说，就是用来模拟上游服务对请求返回指定异常码时，当前的服务是否具备处理能力。 
-  **delay**：非必配项，配置一个 Delay 类型的对象。用来注入延时类故障。通俗一点讲，就是人为模拟上游服务的响应时间，测试在高延迟的情况下，当前的服务是否具备容错容灾的能力。 

###### 延迟与超时

目前针对非luffy登录用户，访问服务的示意为：

```bash
productpage --> reviews v2 --> ratings
               \
                -> details
```

可以通过如下方式，为`ratings`服务注入2秒的延迟：

```bash
$ cat virtualservice-ratings-2s-delay.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
  namespace: bookinfo
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 100
        fixedDelay: 2s
    route:
    - destination:
        host: ratings

$ kubectl apply -f virtualservice-ratings-2s-delay.yaml
# 再次访问http://www.bookinfo.com/productpage，可以明显感觉2s的延迟
```

可以查看对应的envoy的配置：

```bash
$ istioctl pc r ratings-v1-556cfbd589-89ml4.bookinfo --name 9080 -ojson
```



此时的调用为:

```bash
productpage --> reviews v2 -（延迟2秒）-> ratings
               \
                -> details
```



 此时，为reviews服务添加请求超时时间：

```bash
$ kubectl -n bookinfo edit vs reviews
...
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: luffy
    route:
    - destination:
        host: reviews
        subset: v3
  - route:
    - destination:
        host: reviews
        subset: v2
    timeout: 0.5s
...

```

此使的调用关系为：

```bash
productpage -（0.5秒超时）-> reviews v2 -（延迟2秒）-> ratings
               \
                -> details
```

此时，如果使用非luffy用户，则会出现只延迟，不会失败的情况。

删除延迟：

```bash
$ kubectl -n bookinfo delete vs ratings
```



###### 状态码

```bash
$ cat virtualservice-details-aborted.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: details
  namespace: bookinfo
spec:
  hosts:
  - details
  http:
  - fault:
      abort:
        percentage:
          value: 50
        httpStatus: 500
    route:
    - destination:
        host: details

$ kubectl apply -f virtualservice-details-aborted.yaml

# 再次刷新查看details的状态，查看productpage的日志
$ kubectl -n bookinfo logs -f $(kubectl -n bookinfo get po -l app=productpage -ojsonpath='{.items[0].metadata.name}') -c istio-proxy
[2020-11-09T09:00:16.020Z] "GET /details/0 HTTP/1.1" 500 FI "-" "-" 0 18 0 - "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.183 Safari/537.36" "f0387bb6-a445-922c-89ab-689dfbf548f8" "details:9080" "-" - - 10.111.67.169:9080 10.244.0.52:56552 - -

```

