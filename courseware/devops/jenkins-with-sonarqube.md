##### 集成sonarQube实现代码扫描

Sonar可以从以下七个维度检测代码质量，而作为开发人员至少需要处理前5种代码质量问题。

1. 不遵循代码标准
   sonar可以通过PMD,CheckStyle,Findbugs等等代码规则检测工具规范代码编写。
2. 潜在的缺陷
   sonar可以通过PMD,CheckStyle,Findbugs等等代码规则检测工具检 测出潜在的缺陷。
3. 糟糕的复杂度分布
   文件、类、方法等，如果复杂度过高将难以改变，这会使得开发人员 难以理解它们, 且如果没有自动化的单元测试，对于程序中的任何组件的改变都将可能导致需要全面的回归测试。
4. 重复
   显然程序中包含大量复制粘贴的代码是质量低下的，sonar可以展示 源码中重复严重的地方。
5. 注释不足或者过多
   没有注释将使代码可读性变差，特别是当不可避免地出现人员变动 时，程序的可读性将大幅下降 而过多的注释又会使得开发人员将精力过多地花费在阅读注释上，亦违背初衷。
6. 缺乏单元测试
   sonar可以很方便地统计并展示单元测试覆盖率。
7. 糟糕的设计
   通过sonar可以找出循环，展示包与包、类与类之间的相互依赖关系，可以检测自定义的架构规则 通过sonar可以管理第三方的jar包，可以利用LCOM4检测单个任务规则的应用情况， 检测耦合。

###### sonarqube架构简介

![](images\sonarqube.webp)

1. CS架构
   - sonarqube scanner
   - sonarqube server
2. SonarQube Scanner 扫描仪在本地执行代码扫描任务
3. 执行完后，将分析报告被发送到SonarQube服务器进行处理
4. SonarQube服务器处理和存储分析报告导致SonarQube数据库，并显示结果在UI中

###### sonarqube on kubernetes环境搭建

1. 资源文件准备

`sonar/sonar.yaml`

- 和gitlab共享postgres数据库
- 使用ingress地址 `sonar.luffy.com` 进行访问
- 使用initContainers进行系统参数调整

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sonarqube
  namespace: jenkins
  labels:
    app: sonarqube
spec:
  ports:
  - name: sonarqube
    port: 9000
    targetPort: 9000
    protocol: TCP
  selector:
    app: sonarqube
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: jenkins
  name: sonarqube
  labels:
    app: sonarqube
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sonarqube
  template:
    metadata:
      labels:
        app: sonarqube
    spec:
      initContainers:
      - command:
        - /sbin/sysctl
        - -w
        - vm.max_map_count=262144
        image: alpine:3.6
        imagePullPolicy: IfNotPresent
        name: elasticsearch-logging-init
        resources: {}
        securityContext:
          privileged: true
      containers:
      - name: sonarqube
        image: sonarqube:7.9-community
        ports:
        - containerPort: 9000
        env:
        - name: SONARQUBE_JDBC_USERNAME
          valueFrom:
            secretKeyRef:
              name: gitlab-secret
              key: postgres.user.root
        - name: SONARQUBE_JDBC_PASSWORD
          valueFrom:
            secretKeyRef:
              name: gitlab-secret
              key: postgres.pwd.root
        - name: SONARQUBE_JDBC_URL
          value: "jdbc:postgresql://postgres:5432/sonar"
        livenessProbe:
          httpGet:
            path: /sessions/new
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /sessions/new
            port: 9000
          initialDelaySeconds: 60
          periodSeconds: 30
          failureThreshold: 6
        resources:
          limits:
            cpu: 2000m
            memory: 4096Mi
          requests:
            cpu: 300m
            memory: 512Mi
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sonarqube
  namespace: jenkins
spec:
  rules:
  - host: sonar.luffy.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: sonarqube
            port:
              number: 9000

```

2. sonarqube服务端安装

   ```bash
   # 创建sonar数据库
   $ kubectl -n jenkins exec -ti postgres-5859dc6f58-mgqz9 bash
   #/ psql 
   # create database sonar;
   
   ## 创建sonarqube服务器
   $ kubectl create -f sonar.yaml
   
   ## 配置本地hosts解析
   172.21.51.143 sonar.luffy.com
   
   ## 访问sonarqube，初始用户名密码为 admin/admin
   $ curl http://sonar.luffy.com
   
   ```

3. sonar-scanner的安装

   下载地址： https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.2.0.1873-linux.zip。该地址比较慢，可以在网盘下载（https://pan.baidu.com/s/1SiEhWyHikTiKl5lEMX1tJg 提取码: tqb9）。

4. 演示sonar代码扫描功能

   - 在项目根目录中准备配置文件 **sonar-project.properties** 

     ```bash
     sonar.projectKey=myblog
     sonar.projectName=myblog
     # if you want disabled the DTD verification for a proxy problem for example, true by default
     sonar.coverage.dtdVerification=false
     # JUnit like test report, default value is test.xml
     sonar.sources=blog,myblog
     
     ```

   - 配置sonarqube服务器地址

     由于sonar-scanner需要将扫描结果上报给sonarqube服务器做质量分析，因此我们需要在sonar-scanner中配置sonarqube的服务器地址：

     在集群宿主机中测试，先配置一下hosts文件，然后配置sonar的地址：

     ```bash
     $ cat /etc/hosts
     172.21.51.143  sonar.luffy.com
     
     $ cat sonar-scanner/conf/sonar-scanner.properties
     #----- Default SonarQube server
     #sonar.host.url=http://localhost:9000
     sonar.host.url=http://sonar.luffy.com
     #----- Default source code encoding
     #sonar.sourceEncoding=UTF-8
     
     ```

  ```bash
   
# 为了使所有的pod都可以通过`sonar.luffy.com`访问，可以配置coredns的静态解析
$ kubectl -n kube-system edit cm coredns 
...
             hosts {
                 172.21.51.143 jenkins.luffy.com gitlab.luffy.com sonar.luffy.com
                 fallthrough
          }
  ```

   - 执行扫描

     ```bash
     ## 在项目的根目录下执行
     $ /opt/sonar-scanner-4.2.0.1873-linux/bin/sonar-scanner  -X 
     
     ```

   - sonarqube界面查看结果

     登录sonarqube界面查看结果，Quality Gates说明


java项目的配置文件通常格式为：

```bash
sonar.projectKey=eureka-cluster
sonar.projectName=eureka-cluster
# if you want disabled the DTD verification for a proxy problem for example, true by default
# JUnit like test report, default value is test.xml
sonar.sources=src/main/java
sonar.language=java
sonar.tests=src/test/java
sonar.java.binaries=target/classes
```

###### 插件安装及配置

1. 集成到tools容器中

   由于我们的代码拉取、构建任务均是在tools容器中进行，因此我们需要把scanner集成到我们的tools容器中，又因为scanner是一个cli客户端，因此我们直接把包解压好，拷贝到tools容器内部，配置一下PATH路径即可，注意两点：

   - 直接在在tools镜像中配置`http://sonar.luffy.com`

   - 由于tools已经集成了java环境，因此可以直接剔除scanner自带的jre

     - 删掉sonar-scanner/jre目录

     - 修改sonar-scanner/bin/sonar-scanner

       `use_embedded_jre=false`

   ```bash
   $ cd tools
   $ cp -r /opt/sonar-scanner-4.2.0.1873-linux/ sonar-scanner
   ## sonar配置，由于我们是在Pod中使用，也可以直接配置：sonar.host.url=http://sonarqube:9000
   $ cat sonar-scanner/conf/sonar-scanner.properties
   #----- Default SonarQube server
   sonar.host.url=http://sonar.luffy.com
   
   #----- Default source code encoding
   #sonar.sourceEncoding=UTF-8
   
   $ rm -rf sonar-scanner/jre
   $ vi sonar-scanner/bin/sonar-scanner
   ...
   use_embedded_jre=false
   ...
   
   ```

   *Dockerfile*

   `jenkins/custom-images/tools/Dockerfile2`

   ```dockerfile
   FROM alpine:3.13.4
   LABEL maintainer="inspur_lyx@hotmail.com"
   USER root
   
   RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories && \
       apk update && \
       apk add  --no-cache openrc docker git curl tar gcc g++ make \
       bash shadow openjdk8 python2 python2-dev py-pip python3-dev openssl-dev libffi-dev \
       libstdc++ harfbuzz nss freetype ttf-freefont && \
       mkdir -p /root/.kube && \
       usermod -a -G docker root
   
   COPY config /root/.kube/
   
   
   RUN rm -rf /var/cache/apk/*
   
   #-----------------安装 kubectl--------------------#
   COPY kubectl /usr/local/bin/
   RUN chmod +x /usr/local/bin/kubectl
   # ------------------------------------------------#
   
   #---------------安装 sonar-scanner-----------------#
   COPY sonar-scanner /usr/lib/sonar-scanner
   RUN ln -s /usr/lib/sonar-scanner/bin/sonar-scanner /usr/local/bin/sonar-scanner && chmod +x /usr/local/bin/sonar-scanner
   ENV SONAR_RUNNER_HOME=/usr/lib/sonar-scanner
   # ------------------------------------------------#
   
   ```

重新构建镜像，并推送到仓库：

```bash
   $ docker build . -t 172.21.51.143:5000/devops/tools:v2
   $ docker push 172.21.51.143:5000/devops/tools:v2
   
```

2. 修改Jenkins PodTemplate

   为了在新的构建任务中可以拉取v2版本的tools镜像，需要更新PodTemplate

3. 安装并配置sonar插件

   由于sonarqube的扫描的结果需要进行Quality Gates的检测，那么我们在容器中执行完代码扫描任务后，如何知道本次扫描是否通过了Quality Gates，那么就需要借助于sonarqube实现的jenkins的插件。

   - 安装插件

     插件中心搜索sonarqube，直接安装

   - 配置插件

     系统管理->系统配置-> **SonarQube servers** ->Add SonarQube

     - Name：sonarqube

     - Server URL：http://sonar.luffy.com

     - Server authentication token

       ① 登录sonarqube -> My Account -> Security -> Generate Token

       ② 登录Jenkins，添加全局凭据，类型为Secret text

   - 如何在jenkinsfile中使用

     我们在 https://jenkins.io/doc/pipeline/steps/sonar/ 官方介绍中可以看到：

     


###### Jenkinsfile集成sonarqube演示

`jenkins/pipelines/p9.yaml`

```groovy
pipeline {
    agent { label 'jnlp-slave'}
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        timeout(time: 20, unit: 'MINUTES')
        gitLabConnection('gitlab')
    }

    environment {
        IMAGE_REPO = "172.21.51.143:5000/myblog"
        DINGTALK_CREDS = credentials('dingTalk')
        TAB_STR = "\n                    \n&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"
    }

    stages {
        stage('git-log') {
            steps {
                script{
                    sh "git log --oneline -n 1 > gitlog.file"
                    env.GIT_LOG = readFile("gitlog.file").trim()
                }
                sh 'printenv'
            }
        }        
        stage('checkout') {
            steps {
                container('tools') {
                    checkout scm
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS = env.STAGE_NAME + "√..." + env.TAB_STR
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
                            withSonarQubeEnv('sonarqube') {
                                sh 'sonar-scanner -X'
                                sleep 3
                            }
                            script {
                                timeout(1) {
                                    def qg = waitForQualityGate('sonarqube')
                                    if (qg.status != 'OK') {
                                        error "未通过Sonarqube的代码质量阈检查，请及时修改！failure: ${qg.status}"
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
        stage('build-image') {
            steps {
                container('tools') {
                    retry(2) { sh 'docker build . -t ${IMAGE_REPO}:${GIT_COMMIT}'}
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('push-image') {
            steps {
                container('tools') {
                    retry(2) { sh 'docker push ${IMAGE_REPO}:${GIT_COMMIT}'}
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
        stage('deploy') {
            steps {
                container('tools') {
                    sh "sed -i 's#{{IMAGE_URL}}#${IMAGE_REPO}:${GIT_COMMIT}#g' manifests/*"
                    timeout(time: 1, unit: 'MINUTES') {
                        sh "kubectl apply -f manifests/"
                    }
                }
                updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                script{
                    env.BUILD_TASKS += env.STAGE_NAME + "√..." + env.TAB_STR
                }
            }
        }
    }

    post {
        success { 
           container('tools') {
              echo 'Congratulations!'
              sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS_PSW}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😄👍 构建成功 👍😄  \n**项目名称**：luffy  \n**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}   \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
               """ 
           }
        }
        failure {
           container('tools') {
              echo 'Oh no!'
              sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS_PSW}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😖❌ 构建失败 ❌😖  \n**项目名称**：luffy  \n**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}  \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${BUILD_TASKS}"
                        }
                    }'
               """
           }
        }
        always { 
            echo 'I will always say Hello again!'
        }
    }
}

```

若Jenkins执行任务过程中sonarqube端报类似下图的错：
![](images\sonar-scanner-err.png)

则需要在sonarqube服务端进行如下配置，添加一个webhook：
![](images\fix-sonar-scanner-pending-err.png)