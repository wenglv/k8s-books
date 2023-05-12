##### 微服务网关

###### 为什么需要网关

在微服务框架中，每个对外服务都是独立部署的，对外的api或者服务地址都不是不尽相同的。对于内部而言，很简单，通过注册中心自动感知即可。但我们大部分情况下，服务都是提供给外部系统进行调用的，不可能同享一个注册中心。同时一般上内部的微服务都是在内网的，和外界是不连通的。而且，就算我们每个微服务对外开放，对于调用者而言，调用不同的服务的地址或者参数也是不尽相同的，这样就会造成消费者客户端的复杂性，同时想想，可能微服务可能是不同的技术栈实现的，有的是`http`、`rpc`或者`websocket`等等，也会进一步加大客户端的调用难度。所以，**一般上都有会有个api网关，根据请求的url不同，路由到不同的服务上去，同时入口统一了，还能进行统一的身份鉴权、日志记录、分流等操作**。 

![](images\before-gateway.png)

###### 网关的功能

- 减少api请求次数
- 限流
- 缓存
- 统一认证
- 降低微服务的复杂度
- 支持混合通信协议(前端只和api通信，其他的由网关调用)
- ...

![](images\gateway-zuul.png)

###### Zuul实践

新建模块，gateway-zuul,(spring cloud)

pom.xml中需要引入zuul和eureka服务发现的依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.luffy</groupId>
    <artifactId>gateway-zuul</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>gateway-zuul</name>
    <description>gateway-zuul</description>

    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Hoxton.SR9</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <finalName>${project.artifactId}</finalName><!--打jar包去掉版本号-->
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>

```



启动类添加注解

```java
package com.luffy.gatewayzuul;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;

@SpringBootApplication
@EnableZuulProxy
@EnableDiscoveryClient
public class GatewayZuulApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayZuulApplication.class, args);
    }

}

```



配置文件：

```yaml
server:
  port: 10000

spring:
  application:
    name: gateway-zuul

eureka:
  client:
    serviceUrl:
      defaultZone: ${EUREKA_SERVER:http://admin:admin@localhost:8761/eureka/}
  instance:
    instance-id: ${eureka.instance.hostname}:${server.port}
    prefer-ip-address: true
    hostname: ${INSTANCE_HOSTNAME:gateway-zuul}
    #eureka客户端需要多长时间发送心跳给eureka服务器，表明他仍然或者，默认30秒
    lease-renewal-interval-in-seconds: 2
    #eureka服务器在接受到实力的最后一次发出的心跳后，需要等待多久才可以将此实力删除
    lease-expiration-duration-in-seconds: 2
```



启动后，访问：

 http://localhost:10000/bill-service/bill/user/1 

 http://localhost:10000/user-service/user

filter 过滤器

**BILL-SERVICE** =》  [bill-service:7001](http://192.168.136.1:7001/actuator/info) 

通过如下方式，配置短路径：

```yaml
zuul:
  routes:
    user-service: /users/**
    bill-service:
      path: /bill/**
      service-id: bill-service
```

```bash
http://localhost:10000/users/user/1
                                                 --->  http://localhost:7000/user/1
http://localhost:10000/user-service/user/1



http://localhost:10000/bill/service/user/2 
                                                 --->http://localhost:7001/service/user/2
http://localhost:10000/bill-service/service/user/2
```





zuul如何指定对外暴漏api的path，如：

所有的api都是这样：`http://zuul-host:zuul-port/apis/`，可以添加`zuul.prefix：/apis`



配置一下配置文件

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

可以访问到zuul的route列表， http://localhost:10000/actuator/routes/ ，添加details可以访问到详细信息

```json
{
"/apis/users/**": "user-service",
"/apis/bill/**": "bill-service",
"/apis/bill-service/**": "bill-service",
"/apis/user-service/**": "user-service"
}
```

> 提交代码到代码仓库