##### SpringBoot Admin监控

新建项目，`springboot-admin`

`pom.xml`

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
    <artifactId>springboot-admin</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>springboot-admin</name>
    <description>Demo project for Spring Boot</description>
    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Hoxton.SR9</spring-cloud.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-starter-server</artifactId>
            <version>2.2.1</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
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
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

配置文件

```yaml
server:
  port: 8769

spring:
  application:
    name: springboot-admin

eureka:
  client:
    serviceUrl:
      defaultZone: ${EUREKA_SERVER:http://admin:admin@localhost:8761/eureka/}
  instance:
    instance-id: ${eureka.instance.hostname}:${server.port}
    prefer-ip-address: true
    hostname: ${INSTANCE_HOSTNAME:springboot-admin}
    #eureka客户端需要多长时间发送心跳给eureka服务器，表明他仍然或者，默认30秒
    lease-renewal-interval-in-seconds: 2
    #eureka服务器在接受到实力的最后一次发出的心跳后，需要等待多久才可以将此实力删除
    lease-expiration-duration-in-seconds: 2
```

启动类

```java
package com.luffy.springbootadmin;

import de.codecentric.boot.admin.server.config.EnableAdminServer;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
@EnableAdminServer
public class SpringbootAdminApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringbootAdminApplication.class, args);
    }

}
```



客户端，所有注册到eureka的服务，添加依赖即可

```xml
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-starter-client</artifactId>
            <version>2.2.1</version>
        </dependency>
```

