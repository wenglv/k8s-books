#### 微服务间调用

##### 服务提供者

前面已经将用户服务注册到了eureka注册中心，但是还没有暴漏任何API给服务消费者调用。

新建controller类：

```java
package com.luffy.userservice.controller;


import com.luffy.userservice.entity.User;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

import java.util.Random;

@RestController
public class UserController {

    @GetMapping("/user")
    public String getUserService(){
        return "this is user-service";
    }

    @GetMapping("/user-nums")
    public Integer getUserNums(){
        return new Random().nextInt(100);
    }

    //{"id": 123, "name": "张三", "age": 20, "sex": "male"}
    @GetMapping("/user/{id}")
    public User getUserInfo(@PathVariable("id") int id){
        User user = new User();
        user.setId(id);
        user.setAge(20);
        user.setName("zhangsan");
        user.setSex("male");
        return user;
    }
}
```

实体类User.java

```java
package com.luffy.userservice.entity;

public class User {
    private int id;
    private String name;
    private int age;
    private String sex;

    public int getAge() {
        return age;
    }

    public int getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public String getSex() {
        return sex;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public void setId(int id) {
        this.id = id;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setSex(String sex) {
        this.sex = sex;
    }
}
```

`application.yml`

增加从环境变量中读取`EUREKA_SERVER`和`EUREKA_INSTANCE_HOSTNAME`配置

```yaml
server:
  port: 7000
eureka:
  client:
    serviceUrl:
      defaultZone: ${EUREKA_SERVER:http://admin:admin@localhost:8761/eureka/}
  instance:
    instance-id: ${eureka.instance.hostname}:${server.port}
    prefer-ip-address: true
    hostname: ${INSTANCE_HOSTNAME:user-service}
spring:
  application:
    name: user-service
```



###### CICD持续交付服务提供者

`deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: {{NAMESPACE}}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: {{IMAGE_URL}}
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 7000
          resources:
            requests:
              memory: 400Mi
              cpu: 50m
            limits:
              memory: 2Gi
              cpu: 2000m
          env:
            - name: EUREKA_SERVER
              value: "http://admin:admin@eureka-cluster-0.eureka:8761/eureka/,http://admin:admin@eureka-cluster-1.eureka:8761/eureka/,http://admin:admin@eureka-cluster-2.eureka:8761/eureka/"
            - name: INSTANCE_HOSTNAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
```

`service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: {{NAMESPACE}}
spec:
  ports:
    - port: 7000
      protocol: TCP
      targetPort: 7000
  selector:
    app: user-service
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

`ingress.yaml`

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: user-service
  namespace: {{NAMESPACE}}
spec:
  rules:
    - host: {{INGRESS_USER_SERVICE}}
      http:
        paths:
          - backend:
              serviceName: user-service
              servicePort: 7000
            path: /
status:
  loadBalancer: {}
```

ingress配置：

```bash
$ kubectl -n dev edit configmap devops-config
...
data:
  INGRESS_MYBLOG: blog-dev.luffy.com
  INGRESS_SPRINGBOOTDEMO: springboot-dev.luffy.com
  INGRESS_USER_SERVICE: user-service-dev.luffy.com
  NAMESPACE: dev
...
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
        IMAGE_REPO = "172.21.51.143:5000/spring-cloud/user-service"
        IMAGE_CREDENTIAL = "credential-registry"
        DINGTALK_CREDS = credentials('dingTalk')
        PROJECT = "user-service"
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
                    	devops.deploy("manifests",true,"manifests/deployment.yaml").start()
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

`pom.xml`

```xml
<finalName>${project.artifactId}</finalName>
```

`Dockerfile`

```dockerfile
FROM openjdk:8-jdk-alpine
COPY target/user-service.jar app.jar
ENV JAVA_OPTS=""
CMD [ "sh", "-c", "java $JAVA_OPTS -jar /app.jar" ]
```

`sonar-project.properties`

```properties
sonar.projectKey=user-service
sonar.projectName=user-service
# if you want disabled the DTD verification for a proxy problem for example, true by default
# JUnit like test report, default value is test.xml
sonar.sources=src/main/java
sonar.language=java
sonar.tests=src/test/java
sonar.java.binaries=target/classes
```



创建user-service项目，提交代码：

```bash
git init
git remote add origin http://gitlab.luffy.com/luffy-spring-cloud/user-service.git
git add .
git commit -m "Initial commit"
git push -u origin master

# 提交到develop分支
git checkout -b develop
git push -u origin develop
```

创建Jenkins任务，测试自动部署

访问`http://user-service-dev.luffy.com/` 验证