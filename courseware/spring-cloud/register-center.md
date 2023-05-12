#### Spring Cloud开发、交付实践

https://spring.io/projects/spring-cloud#overview

1、Netflix是一家做视频的网站，可以这么说该网站上的美剧应该是最火的。

2、Netflix是一家没有CTO的公司，正是这样的组织架构能使产品与技术无缝的沟通，从而能快速迭代出更优秀的产品。在当时软件敏捷开发中，Netflix的更新速度不亚于当年的微信后台变更，虽然微信比Netflix迟发展，但是当年微信的灰度发布和敏捷开发应该算是业界最猛的。

3、Netflix由于做视频的原因，访问量非常的大，从而促使其技术快速的发展在背后支撑着，也正是如此，Netflix开始把整体的系统往微服务上迁移。

4、Netflix的微服务做的不是最早的，但是确是最大规模的在生产级别微服务的尝试。也正是这种大规模的生产级别尝试，在服务器运维上依托AWS云。当然AWS云同样受益于Netflix的大规模业务不断的壮大。

5、Netflix的微服务大规模的应用，在技术上毫无保留的把一整套微服务架构核心技术栈开源了出来，叫做Netflix OSS，也正是如此，在技术上依靠开源社区的力量不断的壮大。

6、Spring Cloud是构建微服务的核心，而Spring Cloud是基于Spring Boot来开发的。

7、Pivotal在Netflix开源的一整套核心技术产品线的同时，做了一系列的封装，就变成了Spring Cloud；虽然Spring Cloud到现在为止不只有Netflix提供的方案可以集成，还有很多方案，但Netflix是最成熟的。



> 本课程基于SpringBoot 2.3.6.RELEASE 和Spring Cloud  Hoxton.SR9 版本

#### 微服务场景

开发APP，提供个人的花呗账单管理。

- 注册、登录、账单查询 
- 用户服务，账单管理服务

![](images\demo-project.png)

![](images\arch.png)

#### Eureka服务注册中心

在`SpringCloud`体系中，我们知道服务之间的调用是通过`http`协议进行调用的。而注册中心的主要目的就是维护这些服务的服务列表。

 https://docs.spring.io/spring-cloud-netflix/docs/3.0.3/reference/html/ 

##### 新建项目

![](images\new-eureka.jpg)

pom中引入spring-cloud的依赖：

https://spring.io/projects/spring-cloud#overview

```xml
<properties>
    <spring.cloud-version>Hoxton.SR9</spring.cloud-version>
</properties>
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring.cloud-version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

引入eureka-server的依赖：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
```



##### 启动eureka服务

https://docs.spring.io/spring-cloud-netflix/docs/2.2.5.RELEASE/reference/html/#spring-cloud-eureka-server-standalone-mode 

application.yml 

```yaml
server:
  port: 8761
  
eureka:
  client:
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
    register-with-eureka: false
    fetch-registry: false
  instance:
    hostname: localhost
```

启动类：

```java
package com.luffy.eureka;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```



启动访问localhost:8761测试



创建spring cloud项目三部曲：

- 引入依赖包
- 修改application.yml配置文件
- 启动类添加注解





##### eureka认证

没有认证，不安全，添加认证：

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
```



application.yml

```yaml
server:
  port: 8761
eureka:
  client:
    service-url:
      defaultZone: http://${spring.security.user.name}:${spring.security.user.password}@${eureka.instance.hostname}:${server.port}/eureka/
    register-with-eureka: false
    fetch-registry: false
  instance:
    hostname: localhost
spring:
  security:
    user:
      name: ${EUREKA_USER:admin}
      password: ${EUREKA_PASS:admin}
  application:
    name: eureka
```





##### 注册服务到eureka

新建项目，user-service（选择Spring Cloud依赖和SpringBoot Web依赖），用来提供用户查询功能。



三部曲：

- pom.xml，并添加依赖
- 创建application.yml配置文件
- 创建Springboot启动类，并配置注解

`pom.xml`添加：

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```



`application.yml`

```yaml
server:
  port: 7000
eureka:
  client:
    serviceUrl:
      defaultZone: http://${EUREKA_USER:admin}:${EUREKA_PASS:admin}@localhost:8761/eureka/
```



启动类：

```java
package com.luffy.user;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

//注意这里也可使用@EnableEurekaClient
//但由于springcloud是灵活的，注册中心支持eureka、consul、zookeeper等
//若写了具体的注册中心注解，则当替换成其他注册中心时，又需要替换成对应的注解了。
//所以 直接使用@EnableDiscoveryClient 启动发现。
//这样在替换注册中心时，只需要替换相关依赖即可。
@EnableDiscoveryClient
@SpringBootApplication
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```



报错：

```yaml
c.n.d.s.t.d.RetryableEurekaHttpClient    : Request execution failed with message: com.fasterxml.jackson.databind.exc.MismatchedInputException: Root name 'timestamp' does not match expected ('instance') for type [simple type, class com.netflix.appinfo.InstanceInfo]

```



新版本的security默认开启csrf了，关掉，在注册中心新建一个类，继承WebSecurityConfigurerAdapter来关闭 ,> 注意，是在eureka server端关闭。

```java
package com.luffy.eureka;

import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@EnableWebSecurity
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable(); //关闭csrf
        http.authorizeRequests().anyRequest().authenticated().and().httpBasic(); //开启认证
    }
}
```

再次启动发现可以注册，但是地址是

![image-20200915211607173](images\image-20200915211607173.png)



application.yaml

```yaml
server:
  port: 7000
eureka:
  client:
    serviceUrl:
      defaultZone: http://${EUREKA_USER:admin}:${EUREKA_PASS:admin}@localhost:8761/eureka/
  instance:
    instance-id: ${eureka.instance.hostname}:${server.port}
    prefer-ip-address: true
    hostname: user-service
spring:
  application:
    name: user-service
```



Eurake有一个配置参数eureka.server.renewalPercentThreshold，定义了renews 和renews threshold的比值，默认值为0.85。当server在15分钟内，比值低于percent，即少了15%的微服务心跳，server会进入自我保护状态 

默认情况下，如果`Eureka Server`在一定时间内没有接收到某个微服务实例的心跳，`Eureka Server`将会注销该实例（默认90秒）。但是当网络分区故障发生时，微服务与Eureka Server之间无法正常通信，这就可能变得非常危险了，因为微服务本身是健康的，此时本不应该注销这个微服务。

`Eureka Server`通过“自我保护模式”来解决这个问题，当`Eureka Server`节点在短时间内丢失过多客户端时（可能发生了网络分区故障），那么这个节点就会进入自我保护模式。一旦进入该模式，`Eureka Server`就会保护服务注册表中的信息，不再删除服务注册表中的数据（也就是不会注销任何微服务）。当网络故障恢复后，该`Eureka Server`节点会自动退出自我保护模式。

**自我保护模式是一种对网络异常的安全保护措施。使用自我保护模式，而让Eureka集群更加的健壮、稳定。**

开发阶段可以通过配置：`eureka.server.enable-self-preservation=false`关闭自我保护模式。

**生产阶段，理应以默认值进行配置。**

至于具体具体的配置参数，可至官网查看：http://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html#_appendix_compendium_of_configuration_properties



##### 高可用

高可用：

- 优先保证可用性
- 各个节点都是平等的，1个节点挂掉不会影响正常节点的工作，剩余的节点依然可以提供注册和查询服务
- 在向某个Eureka注册时如果发现连接失败，则会自动切换至其它节点，只要有一台Eureka还在，就能保证注册服务可用(保证可用性)

注意点：

- 多实例的话eureka.instance.instance-id需要保持不一样，否则会当成同一个
- eureka.instance.hostname要与defaultZone里的地址保持一致
- 各个eureka的spring.application.name相同

![](images\eureka-cluster.jpg)

拷贝`eureka`服务，分别命名`eureka-ha-peer1`和`eureka-ha-peer2`

修改模块的`pom.xml`

```xml
<artifactId>eureka-ha-peer1</artifactId>
```

修改配置文件`application.yml`，注意集群服务，需要各个eureka的spring.application.name相同

```yaml
server:
  port: ${EUREKA_PORT:8762}
eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_SERVER:http://${spring.security.user.name}:${spring.security.user.password}@peer1:8762/eureka/,http://${spring.security.user.name}:${spring.security.user.password}@peer2:8763/eureka/}
    fetch-registry: true
  instance:
    instance-id: ${eureka.instance.hostname}:${server.port}
    hostname: peer1
spring:
  security:
    user:
      name: ${EUREKA_USER:admin}
      password: ${EUREKA_PASS:admin}
  application:
    name: eureka-cluster
```

设置hosts文件

```bash
127.0.0.1 peer1 peer2
```





服务提供者若想连接高可用的eureka，需要修改：

```bash
      defaultZone: http://${EUREKA_USER:admin}:${EUREKA_PASS:admin}@peer1:8762/eureka/,http://${EUREKA_USER:admin}:${EUREKA_PASS:admin}@peer2:8763/eureka/

```



##### k8s交付

分析：

高可用互相注册，但是需要知道对方节点的地址。k8s中pod ip是不固定的，如何将高可用的eureka服务使用k8s交付？

- 方案一：创建三个Deployment+三个Service

  ![](images\eureka-ha-deploy.jpg)

  

- 方案二：使用statefulset管理

  ![](images\eureka-sts.jpg)

`eureka-statefulset.yaml`

```yaml
# eureka-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: eureka-cluster
  namespace: dev
spec:
  serviceName: "eureka"
  replicas: 3
  selector:
    matchLabels:
      app: eureka-cluster
  template:
    metadata:
      labels:
        app: eureka-cluster
    spec:
      containers:
        - name: eureka
          image: 172.21.51.143:5000/spring-cloud/eureka-cluster:v1
          ports:
            - containerPort: 8761
          resources:
            requests:
              memory: 400Mi
              cpu: 50m
            limits:
              memory: 2Gi
              cpu: 2000m
          env:
            - name: MY_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: JAVA_OPTS
              value: -XX:+UnlockExperimentalVMOptions
                -XX:+UseCGroupMemoryLimitForHeap
                -XX:MaxRAMFraction=2
                -XX:CICompilerCount=8
                -XX:ActiveProcessorCount=8
                -XX:+UseG1GC
                -XX:+AggressiveOpts
                -XX:+UseFastAccessorMethods
                -XX:+UseStringDeduplication
                -XX:+UseCompressedOops
                -XX:+OptimizeStringConcat
            - name: EUREKA_SERVER
              value: "http://admin:admin@eureka-cluster-0.eureka:8761/eureka/,http://admin:admin@eureka-cluster-1.eureka:8761/eureka/,http://admin:admin@eureka-cluster-2.eureka:8761/eureka/"
            - name: EUREKA_INSTANCE_HOSTNAME
              value: ${MY_POD_NAME}.eureka
            - name: EUREKA_PORT
              value: "8761"
```

`eureka-headless-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: eureka
  namespace: dev
  labels:
    app: eureka
spec:
  ports:
    - port: 8761
      name: eureka
  clusterIP: None
  selector:
    app: eureka-cluster
```



想通过ingress访问eureka，需要使用有头服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: eureka-ingress
  namespace: dev
  labels:
    app: eureka-cluster
spec:
  ports:
    - port: 8761
      name: eureka-cluster
  selector:
    app: eureka-cluster
```



ingress提供访问入口：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: eureka-cluster
  namespace: dev
spec:
  rules:
  - host: eureka-cluster.luffy.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: eureka-ingress
            port:
              number: 8761
```



##### 使用StatefulSet管理有状态服务

使用StatefulSet创建多副本pod的情况：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-statefulset
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
$ kubectl -n spring exec  -ti nginx-statefulset-0 sh
/ # curl nginx-statefulset-2.nginx
```



##### 接入CICD流程

所需的文件:

- Jenkinsfile
- Dockerfile
- manifests/statefulset.yaml,service.yaml
- sonar-project.properties

在pom.xml中重写jar包名称：

```xml
<finalName>${project.artifactId}</finalName>
```

`Dockerfile`

```dockerfile
FROM openjdk:8-jdk-alpine
ADD target/eureka.jar app.jar
ENV JAVA_OPTS=""
CMD [ "sh", "-c", "java $JAVA_OPTS -jar /app.jar" ]
```



`Jenkinsfile`

```groovy
@Library('luffy-devops') _

pipeline {
    agent { label 'jnlp-slave'}
    options {
		timeout(time: 20, unit: 'MINUTES')
		gitLabConnection('gitlab')
	}
    environment {
        IMAGE_REPO = "172.21.51.143:5000/spring-cloud/eureka-cluster"
        IMAGE_CREDENTIAL = "credential-registry"
        DINGTALK_CREDS = credentials('dingTalk')
        PROJECT = "eureka-cluster"
    }
    stages {
        stage('checkout') {
            steps {
                container('tools') {
                    checkout scm
                }
            }
        }
        stage('mvn-package') {
            steps {
                container('tools') {
                    script{
                        sh 'mvn clean package'
                    }
                }
            }
        }
        stage('CI'){
            failFast true
            parallel {
                stage('Unit Test') {
                    steps {
                        echo "Unit Test Stage Skip..."
                    }
                }
                stage('Code Scan') {
                    steps {
                        container('tools') {
                            script {
                               devops.scan().start()
                            }
                        }
                    }
                }
            }
        }

        stage('docker-image') {
            steps {
                container('tools') {
                    script{
                        devops.docker(
                            "${IMAGE_REPO}",
                            "${GIT_COMMIT}",
                            IMAGE_CREDENTIAL
                        ).build().push()
                    }
                }
            }
        }
        stage('deploy') {
            steps {
                container('tools') {
                    script{
                    	devops.deploy("manifests",false,"manifests/statefulset.yaml").start()
                    }
                }
            }
        }
    }
    post {
        success {
            script{
                devops.notificationSuccess(PROJECT,"dingTalk")
            }
        }
        failure {
            script{
                devops.notificationFailure(PROJECT,"dingTalk")
            }
        }
    }
}
```

`sonar-project.properties`

```properties
sonar.projectKey=eureka-cluster
sonar.projectName=eureka-cluster
# if you want disabled the DTD verification for a proxy problem for example, true by default
# JUnit like test report, default value is test.xml
sonar.sources=src/main/java
sonar.language=java
sonar.tests=src/test/java
sonar.java.binaries=target/classes
```



模板化k8s资源清单：

```bash
# eureka-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: eureka-cluster
  namespace: {{NAMESPACE}}
spec:
  serviceName: "eureka"
  replicas: 3
  selector:
    matchLabels:
      app: eureka-cluster
  template:
    metadata:
      labels:
        app: eureka-cluster
    spec:
      containers:
        - name: eureka
          image: {{IMAGE_URL}}
...
```



维护新组件的ingress:

```bash
$ kubectl -n dev edit configmap devops-config
...
  INGRESS_EUREKA: eureka.luffy.com
...
```





部署k8s集群时，将eureka的集群地址通过参数的形式传递到pod内部，因此本地开发时，直接按照单点模式进行：

```yaml
server:
  port: ${EUREKA_PORT:8761}
eureka:
  client:
    service-url:
      defaultZone: ${EUREKA_SERVER:http://${spring.security.user.name}:${spring.security.user.password}@localhost:8761/eureka/}
    fetch-registry: true
    register-with-eureka: true
  instance:
    instance-id: ${eureka.instance.hostname}:${server.port}
    hostname: ${EUREKA_INSTANCE_HOSTNAME:localhost}
    prefer-ip-address: true
spring:
  security:
    user:
      name: ${EUREKA_USER:admin}
      password: ${EUREKA_PASS:admin}
  application:
    name: eureka-cluster
```

提交项目：

创建develop分支，CICD部署开发环境

