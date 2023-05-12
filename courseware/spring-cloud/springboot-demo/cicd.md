
##### Springboot服务镜像制作

通过`mvn package`命令拿到服务的jar包后，我们可以使用如下命令启动服务：

```bash
$ java -jar springboot-demo-0.0.1-SNAPSHOT.jar
```

因此，需要准备Dockerfile来构建镜像：

```dockerfile
FROM openjdk:8-jdk-alpine
COPY target/springboot-demo-0.0.1-SNAPSHOT.jar app.jar
CMD [ "sh", "-c", "java -jar /app.jar" ]
```

我们可以为构建出的镜像指定名称：

```xml
    <build>
        <finalName>${project.artifactId}</finalName><!--打jar包去掉版本号-->
    ...
```

`Dockerfile`对应修改：

```dockerfile
FROM openjdk:8-jdk-alpine
COPY target/springboot-demo.jar app.jar
CMD [ "sh", "-c", "java -jar /app.jar" ]
```



执行镜像构建，验证服务启动是否正常：

```bash
$ docker build . -t springboot-demo:v1 -f Dockerfile

$ docker run -d --name springboot-demo -p 8080:8080 springboot-demo:v1

$ curl localhost:8080
```



##### 接入CICD流程

之前已经实现了shared-library，并且把python项目接入到了CICD 流程中。因此，可以直接使用已有的流程，把spring boot项目接入进去。

- `Jenkinsfile`
- `sonar-project.properties`
- `manifests/deployment.yaml`
- `manifests/service.yaml`
- `manifests/ingress.yaml`
- `configmap/devops-config`

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
        IMAGE_REPO = "172.21.51.143:5000/spring-cloud/springboot-demo"
        IMAGE_CREDENTIAL = "credential-registry"
        DINGTALK_CREDS = credentials('dingTalk')
        PROJECT = "springboot-demo"
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



`sonar-project.properties`

```java
sonar.projectKey=springboot-demo
sonar.projectName=springboot-demo
# if you want disabled the DTD verification for a proxy problem for example, true by default
# JUnit like test report, default value is test.xml
sonar.sources=src/main/java
sonar.language=java
sonar.tests=src/test/java
sonar.java.binaries=target/classes
```



`manifests/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot-demo
  namespace: {{NAMESPACE}}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: springboot-demo
  template:
    metadata:
      labels:
        app: springboot-demo
    spec:
      containers:
        - name: springboot-demo
          image: {{IMAGE_URL}}
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: 100Mi
              cpu: 50m
            limits:
              memory: 500Mi
              cpu: 100m
          livenessProbe:
            httpGet:
              path: /
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 120
            periodSeconds: 15
            timeoutSeconds: 3
          readinessProbe:
            httpGet:
              path: /
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 120
            timeoutSeconds: 2
            periodSeconds: 15
```



`manifests/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: springboot-demo
  namespace: {{NAMESPACE}}
spec:
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    app: springboot-demo
  sessionAffinity: None
  type: ClusterIP
```



`manifests/ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: springboot-demo
  namespace: {{NAMESPACE}}
spec:
  rules:
  - host: {{INGRESS_SPRINGBOOTDEMO}}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: springboot-demo
            port:
              number: 8080
```

维护`devops-config`的`configmap`，添加`INGRESS_SPRINGBOOTDEMO`配置项：

```bash
$ kubectl -n dev edit cm devops-config
...
data:
  INGRESS_MYBLOG: blog-dev.luffy.com
  INGRESS_SPRINGBOOTDEMO: springboot-dev.luffy.com
  NAMESPACE: dev
...
```





更新Jenkins中的jnlp-slave-pod模板镜像：

```bash
172.21.51.143:5000/devops/tools:v4
```

由于镜像中maven的目录是`/opt/maven-repo`，而slave-pod是执行完任务后会销毁，因此需要将maven的数据目录挂载出来，不然每次构建都会重新拉取所有依赖的jar包：

![](images\maven-repo-vols.jpg)

配置Jenkins流水线：





##### 添加单元测试覆盖率

单元测试这块内容一直没有把覆盖率统计到`sonarqube`端，本节看下怎么样将单元测试的结果及覆盖率展示到Jenkins及`sonarqube`平台中。

为了展示效果，我们先添加一个单元测试文件`HelloControllerTests`：

```java
package com.luffy.springbootdemo;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.web.WebAppConfiguration;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;
import org.springframework.test.web.servlet.result.MockMvcResultHandlers;
import org.springframework.test.web.servlet.result.MockMvcResultMatchers;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

@SpringBootTest
@WebAppConfiguration
public class HelloControllerTests {

    private static final Logger logger = LoggerFactory.getLogger(HelloControllerTests.class);
    @Autowired
    private WebApplicationContext webApplicationContext;

    private MockMvc mockMvc;

    @BeforeEach
    public void setMockMvc() {
        mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext).build();
    }

    @Test
    public void index(){
        try {
            mockMvc.perform(MockMvcRequestBuilders.post("/")
                    .contentType(MediaType.APPLICATION_JSON)
            ).andExpect(MockMvcResultMatchers.status().isOk())
                    .andDo(MockMvcResultHandlers.print());
        }catch (Exception e) {
            e.printStackTrace();
        }

    }

    @Test
    public void rightaway(){
        try {
            mockMvc.perform(MockMvcRequestBuilders.post("/rightaway")
                    .contentType(MediaType.APPLICATION_JSON)
            ).andExpect(MockMvcResultMatchers.status().isOk())
                    .andDo(MockMvcResultHandlers.print());
        }catch (Exception e) {
            e.printStackTrace();
        }

    }

    @Test
    public void sleep(){
        try {
            mockMvc.perform(MockMvcRequestBuilders.post("/sleep")
                    .contentType(MediaType.APPLICATION_JSON)
            ).andExpect(MockMvcResultMatchers.status().isOk())
                    .andDo(MockMvcResultHandlers.print());
        }catch (Exception e) {
            e.printStackTrace();
        }

    }
}
```

`jacoco`：监控JVM中的调用，生成监控结果（默认保存在`jacoco.exec`文件中），然后分析此结果，配合源代码生成覆盖率报告。 

如何引入`jacoco`测试：

```yaml
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <version>0.7.8</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>prepare-agent</goal>
                        </goals>
                        <configuration>
                            <destFile>${project.build.directory}/coverage-reports/jacoco.exec</destFile>
                        </configuration>
                    </execution>
                    <execution>
                        <id>default-report</id>
                        <phase>test</phase>
                        <goals>
                            <goal>report</goal>
                        </goals>
                        <configuration>
                            <dataFile>${project.build.directory}/coverage-reports/jacoco.exec</dataFile>
                            <outputDirectory>${project.reporting.outputDirectory}/jacoco</outputDirectory>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
```

其中：

- `prepare-agent`，会把agent准备好，这样在执行用例的时候，就会使用agent检测到代码执行的过程，通常将结果保存在`jacoco.exec`中
- `report`，分析保存的`jacoco.exec`文件，生成报告

在IDE中添加，观察插件的goal，执行`mvn test`，观察执行过程。



有了上述内容后，如何将结果发布到`sonarqube`中？

提交最新代码，查看`sonarqube`的分析结果。

![](images\sonar-coverage.jpg)

