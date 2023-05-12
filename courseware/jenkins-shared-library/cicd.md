###### library集成触发任务

##### 多环境的CICD自动化实现



###### 实现目标及效果

目前项目存在`develop`和`master`两个分支，Jenkinsfile中配置的都是构建部署到相同的环境，实际的场景中，代码仓库的项目往往不同的分支有不同的作用，我们可以抽象出一个工作流程：

![](images\multi-envs.jpg)

- 开发人员提交代码到develop分支

- Jenkins自动使用develop分支做单测、代码扫描、镜像构建（以commit id为镜像tag）、服务部署到开发环境
- 开发人员使用开发环境自测
- 测试完成后，在gitlab提交merge request请求，将代码合并至master分支
- 需要发版时，在gitlab端基于master分支创建tag（v2.3.0）
- Jenkins自动检测到tag，拉取tag关联的代码做单测、代码扫描、镜像构建（以代码的tag为镜像的tag）、服务部署到测试环境、执行集成测试用例，输出测试报告
- 测试人员进行手动测试
- 上线





###### 实现思路

以myblog项目为例，目前已经具备的是develop分支代码提交后，可以自动实现：

- 单元测试、代码扫描
- 镜像构建
- k8s服务部署
- robot集成用例测试

和上述目标相比，差异点：

1. myblog应用目前只有一套环境，在luffy命名空间中。我们新建两个命名空间：
   - dev，用作部署开发环境
   - test，用作部署集成测试环境
2. 需要根据不同的分支来执行不同的任务，有两种方案实现：
   - develop和master分支使用不同的Jenkinsfile
     - 可行性很差，因为代码合并工作很繁琐
     - 维护成本高，多个分支需要维护多个Jenkinsfile
   - 使用同一套Jenkinsfile，配合library和模板来实现一套Jenkinsfile适配多套环境
     - 改造Jenkinsfile，实现根据分支来选择任务
     - 需要将deploy目录中所有和特定环境绑定的内容模板化
     - 在library中实现根据不同的分支，来替换模板中的内容

###### Jenkinsfile根据分支选择任务

使用when关键字，配合正则表达式，实现分支的过滤选择：

```bash
pipeline {
    agent any
    stages {
        stage('Example Build') {
            steps {
                echo 'Hello World'
            }
        }
        stage('Example Deploy') {
            when {
                expression { BRANCH_NAME ==~ "develop" }
            }
            steps {
                echo 'Deploying to develop env'
            }
        }
    }
}
```

分别在develop和master分支进行验证。



针对本例，可以对Jenkinsfile做如下调整：

```bash
...
        stage('integration test') {
            when {
                expression { BRANCH_NAME ==~ /v.*/ }
            }
            steps {
                container('tools') {
                    script{
                    	devops.robotTest(PROJECT)
                    }
                }
            }
        }
...
```



###### 模板化k8s的资源清单

因为需要使用同一套模板和Jenkinsfile来部署到不同的环境，因此势必要对资源清单进行模板化，前面的内容中只将`deployment.yaml`放到了项目的`manifests`清单目录，此处将部署myblog用到的资源清单均补充进去，包含：

- deployment.yaml
- service.yaml
- ingress.yaml
- configmap.yaml
- secret.yaml

涉及到需要进行模板化的内容包括：

- 镜像地址

- 命名空间

- ingress的域名信息


模板化后的文件：

```bash
$ cat deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myblog
  namespace: {{NAMESPACE}}
spec:
  replicas: 1   #指定Pod副本数
  selector:             #指定Pod的选择器
    matchLabels:
      app: myblog
  template:
    metadata:
      labels:   #给Pod打label
        app: myblog
    spec:
      containers:
      - name: myblog
        image: {{IMAGE_URL}}
        imagePullPolicy: IfNotPresent
        env:
        - name: MYSQL_HOST
          valueFrom:
            configMapKeyRef:
              name: myblog
              key: MYSQL_HOST
        - name: MYSQL_PORT
          valueFrom:
            configMapKeyRef:
              name: myblog
              key: MYSQL_PORT
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: myblog
              key: MYSQL_USER
        - name: MYSQL_PASSWD
          valueFrom:
            secretKeyRef:
              name: myblog
              key: MYSQL_PASSWD
        ports:
        - containerPort: 8002
        resources:
          requests:
            memory: 100Mi
            cpu: 50m
          limits:
            memory: 500Mi
            cpu: 100m
        livenessProbe:
          httpGet:
            path: /blog/index/
            port: 8002
            scheme: HTTP
          initialDelaySeconds: 10  # 容器启动后第一次执行探测是需要等待多少秒
          periodSeconds: 15     # 执行探测的频率
          timeoutSeconds: 2             # 探测超时时间
        readinessProbe: 
          httpGet: 
            path: /blog/index/
            port: 8002
            scheme: HTTP
          initialDelaySeconds: 10 
          timeoutSeconds: 2
          periodSeconds: 15

$ cat configmap.yaml
apiVersion: v1
data:
  MYSQL_HOST: mysql
  MYSQL_PORT: "3306"
kind: ConfigMap
metadata:
  name: myblog
  namespace: {{NAMESPACE}}

$ cat secret.yaml
apiVersion: v1
data:
  MYSQL_PASSWD: MTIzNDU2
  MYSQL_USER: cm9vdA==
kind: Secret
metadata:
  name: myblog
  namespace: {{NAMESPACE}}
type: Opaque

$ cat service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myblog
  namespace: {{NAMESPACE}}
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8002
  selector:
    app: myblog
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}

$ cat ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: myblog
  namespace: {{NAMESPACE}}
spec:
  rules:
  - host: {{INGRESS_MYBLOG}}
    http:
      paths:
      - backend:
          serviceName: myblog
          servicePort: 80
        path: /
status:
  loadBalancer: {}
```



###### 实现library配置替换逻辑

我们需要实现使用相同的模板，做到如下事情：

- 根据代码分支来部署到不同的命名空间
  - develop分支部署到开发环境，使用命名空间 dev
  - v.*部署到测试环境，使用命名空间 test
- 不同环境使用不同的ingress地址来访问
  - 开发环境，`blog-dev.luffy.com`
  - 测试环境，`blog-test.luffy.com`



如何实现？sharedlibrary

所有的逻辑都会经过library这一层，我们具有完全可控权。

前面已经替换过镜像地址了，我们只需要实现如下逻辑：

- 检测当前代码分支，替换命名空间
- 检测当前代码分支，替换Ingress地址

问题来了，如何检测构建的触发是develop分支还是tag分支？

答案是：env.TAG_NAME，由tag分支触发的构建，环境变量中会带有TAG_NAME，且值为gitlab中的tag名称。

做个演示：

使用如下的Jenkinsfile，查看由master分支触发和由tag分支触发，printenv的值有什么不同

```groovy
pipeline {
    agent any
    stages {
        stage('Example Build') {
            steps {
                echo 'Hello World'
                sh 'printenv'
            }
        }
        stage('Example Deploy') {
            when {
                expression { BRANCH_NAME ==~ "develop" }
            }
            steps {
                echo 'Deploying to develop env'
            }
        }
    }
}
```



我们可以选择和替换image镜像地址一样，来执行替换：

```groovy
def tplHandler(){
    sh "sed -i 's#{{IMAGE_URL}}#${env.CURRENT_IMAGE}#g' ${this.resourcePath}/*"
    String namespace = "dev"
    String ingress = "blog-dev.luffy.com"
    if(env.TAG_NAME){
        namespace = "test"
        ingress = "blog-test.luffy.com"
    }
    sh "sed -i 's#{{NAMESPACE}}#${namespace}#g' ${this.resourcePath}/*"
    sh "sed -i 's#{{INGRESS_MYBLOG}}#${ingress}#g' ${this.resourcePath}/*"
}
```

但是我们的library是要为多个项目提供服务的，如果采用上述方式，则每加入一个项目，都需要对library做改动，形成了强依赖。因此需要想一种更优雅的方式来进行替换。

思路：

1. 开发环境和集成测试环境里准备一个configmap，取名为 `devops-config`

2. configmap的内容大致如下：

   - 开发环境

     ```bash
     NAMESPACE=dev
     INGRESS_MYBLOG=blog-dev.luffy.com
     INGRESS_BUSINESS_A=xxx.luffy.com
     ```

   - 测试环境

     ```bash
     NAMESPACE=test
     INGRESS_MYBLOG=blog-test.luffy.com
     ```

3. 约定：configmap的key值，拼接{{KEY}}则为代码中需要替换的模板部分，configmap的该key对应的value，则为该模板要被替换的值的内容。比如：

   ```bash
   NAMESPACE=dev
   INGRESS_MYBLOG=blog-dev.luffy.com
   {{NAMESPACE}} => dev
   {{INGRESS_MYBLOG}} -> blog-dev.luffy.com
   ```

   意思是约定项目的deploy的资源清单中：

   - 所有的`{{NAMESPACE}}`被替换为`dev`
   - 所有的`{{INGRESS_MYBLOG}}`被替换为`blog-dev.luffy.com`

4. 在library的逻辑中，实现读取触发当前构建的代码分支所关联的namespace下的`devops-config`这个configmap，然后遍历里面的值进行模板替换即可。

这样，则以后再有新增的项目，则只需要维护`devops-config`配置文件即可，shared-library则不需要随着项目的增加而进行修改，通过这种方式实现library和具体的项目解耦。

```groovy
def tplHandler(){
    sh "sed -i 's#{{IMAGE_URL}}#${env.CURRENT_IMAGE}#g' ${this.resourcePath}/*"
    String namespace = "dev"
    if(env.TAG_NAME){
        namespace = "test"
    }
    try {
        def configMapData = this.getResource(namespace, "devops-config", "configmap")["data"]
        configMapData.each { k, v ->
            echo "key is ${k}, val is ${v}"
            sh "sed -i 's#{{${k}}}#${v}#g' ${this.resourcePath}/*"
        }
    }catch (Exception exc) {
        echo "failed to get devops-config data,exception: ${exc}."
        throw exc
    }
}
```



###### 准备多环境

1. 创建开发和测试环境的命名空间

   ```bash
   # 
   $ kubectl create namespace dev
   $ kubectl create namespace test
   ```

2. 分别在dev和test命名空间准备mysql数据库。演示功能，因此mysql未作持久化

   ```bash
   $ cat mysql-all.yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: mysql
     namespace: dev
   spec:
     ports:
     - port: 3306
       protocol: TCP
       targetPort: 3306
     selector:
       app: mysql
     type: ClusterIP
   ---
   apiVersion: v1
   kind: Secret
   metadata:
     name: myblog
     namespace: dev
   type: Opaque
   data:
     MYSQL_USER: cm9vdA==
     MYSQL_PASSWD: MTIzNDU2
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: mysql
     namespace: dev
   spec:
     replicas: 1   #指定Pod副本数
     selector:             #指定Pod的选择器
       matchLabels:
         app: mysql
     template:
       metadata:
         labels:   #给Pod打label
           app: mysql
       spec:
         containers:
         - name: mysql
           image: mysql:5.7
           args:
           - --character-set-server=utf8mb4
           - --collation-server=utf8mb4_unicode_ci
           ports:
           - containerPort: 3306
           env:
           - name: MYSQL_ROOT_PASSWORD
             valueFrom:
               secretKeyRef:
                 name: myblog
                 key: MYSQL_PASSWD
           - name: MYSQL_DATABASE
             value: "myblog"
           resources:
             requests:
               memory: 100Mi
               cpu: 50m
             limits:
               memory: 500Mi
               cpu: 100m
           readinessProbe:
             tcpSocket:
               port: 3306
             initialDelaySeconds: 5
             periodSeconds: 10
             
   # 创建开发环境的数据库
   $ kubectl create -f mysql-all.yaml
   
   # 替换dev命名空间，创建测试环境的数据库
   $ sed -i 's/namespace: dev/namespace: test/g' mysql-all.yaml
   $ kubectl create -f mysql-all.yaml
   ```

   

3. 对myblog项目的k8s资源清单模板化改造

   - {{NAMESPACE}}
   - {{INGRESS_MYBLOG}}
   - {{IMAGE_URL}}

4. 初始化开发环境和测试环境的`devops-config`

   ```bash
   # 开发环境
   $ cat devops-config-dev.txt
   NAMESPACE=dev
   INGRESS_MYBLOG=blog-dev.luffy.com
   
   $ kubectl -n dev create configmap devops-config --from-env-file=devops-config-dev.txt
   
   # 测试环境
   $ cat devops-config-test.txt
   NAMESPACE=test
   INGRESS_MYBLOG=blog-test.luffy.com
   
   $ kubectl -n test create configmap devops-config --from-env-file=devops-config-test.txt
   
   ```

5. 提交最新的library代码

6. 提交最新的python-demo项目代码

   ```groovy
   @Library('luffy-devops') _
   
   pipeline {
       agent { label 'jnlp-slave'}
       options {
   		timeout(time: 20, unit: 'MINUTES')
   		gitLabConnection('gitlab')
   	}
       environment {
           IMAGE_REPO = "172.21.51.143:5000/myblog"
           IMAGE_CREDENTIAL = "credential-registry"
           DINGTALK_CREDS = credentials('dingTalk')
           PROJECT = "myblog"
       }
       stages {
           stage('checkout') {
               steps {
                   container('tools') {
                       checkout scm
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
           stage('integration test') {
               when {
                   expression { BRANCH_NAME ==~ /v.*/ }
               }
               steps {
                   container('tools') {
                       script{
                       	devops.robotTest(PROJECT)
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
                   devops.notificationFailed(PROJECT,"dingTalk")
               }
           }
       }
   }
   ```

   

###### 验证多环境自动部署

模拟如下流程：

1. 提交代码到develop分支，观察是否部署到dev的命名空间中，注意，第一次部署，需要执行migrate操作：

   ```bash
    $ kubectl -n dev exec  myblog-9f9f7c8cd-k6tbj python3 manage.py migrate
   ```

2. 配置hosts解析，测试使用`http://blog-dev.luffy.com/blog/index/`进行访问到develop分支最新版本

3. 合并代码至master分支

4. 在gitlab中创建tag，观察是否自动部署至test的命名空间中，且使用`myblog-test.luffy.com/blog/index/`可以访问到最新版本

###### 实现打tag后自动部署

我们发现，打了tag以后，多分支流水线中可以识别到该tag，但是并不会自动部署该tag的代码。因此，我们来使用一个新的插件：Basic Branch Build Strategies Plugin

安装并配置多分支流水线，注意Build strategies 设置：

-  Regular branches
-  Tags
   -  Ignore tags newer than 可以不用设置，不然会默认不自动构建新打的tag
   -  Ignore tags older than 

###### 优化镜像部署逻辑

针对部署到测试环境的代码，由于已经打了tag了，因此，我们期望构建出来的镜像地址可以直接使用代码的tag作为镜像的tag。

思路一：直接在Jenkinsfile调用`devops.docker`时传递tag名称

思路二：在shared-library中，根据`env.TAG_NAME`来判断当前是否是tag分支的构建，若TAG_NAME不为空，则可以在构建镜像时使用TAG_NAME作为镜像的tag

很明显我们更期望使用思路二的方式来实现，因此，需要调整如下逻辑：

```bash
def docker(String repo, String tag, String credentialsId, String dockerfile="Dockerfile", String context="."){
    this.repo = repo
    this.tag = tag
    if(env.TAG_NAME){
        this.tag = env.TAG_NAME
    }
    this.dockerfile = dockerfile
    this.credentialsId = credentialsId
    this.context = context
    this.fullAddress = "${this.repo}:${this.tag}"
    this.isLoggedIn = false
    this.msg = new BuildMessage()
    return this
}
```

提交代码，并进行测试，观察是否使用tag作为镜像标签进行部署。