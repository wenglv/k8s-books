##### library集成镜像构建及推送

需要实现的逻辑点：

- docker build，docker push，docker login
- 账户密码，jenkins凭据，（library中获取凭据内容）
- docker login 172.21.51.143:5000
- try catch

###### 镜像构建逻辑实现

`devops.groovy`

```groovy
/**
 *
 * @param repo, 172.21.51.143:5000/demo/myblog/xxx/
 * @param tag, v1.0
 * @param dockerfile
 * @param credentialsId
 * @param context
 */
def docker(String repo, String tag, String credentialsId, String dockerfile="Dockerfile", String context=".") {
    return new Docker().docker(repo, tag, credentialsId, dockerfile, context)
}
```



`Docker.groovy`

逻辑中需要注意的点：

- 构建和推送镜像，需要登录仓库（需要认证）
- 构建成功或者失败，需要将结果推给gitlab端
- 为了将构建过程推送到钉钉消息中，需要将构建信息统一收集

```bash
package com.luffy.devops

/**
 *
 * @param repo
 * @param tag
 * @param credentialsId
 * @param dockerfile
 * @param context
 * @return
 */
def docker(String repo, String tag, String credentialsId, String dockerfile="Dockerfile", String context="."){
    this.repo = repo
    this.tag = tag
    this.dockerfile = dockerfile
    this.credentialsId = credentialsId
    this.context = context
    this.fullAddress = "${this.repo}:${this.tag}"
    this.isLoggedIn = false
    return this
}


/**
 * build image
 * @return
 */
def build() {
    this.login()
    retry(3) {
        try {
            sh "docker build ${this.context} -t ${this.fullAddress} -f ${this.dockerfile} "
        }catch (Exception exc) {
            throw exc
        }
        return this
    }
}


/**
 * push image
 * @return
 */
def push() {
    this.login()
    retry(3) {
        try {
            sh "docker push ${this.fullAddress}"
        }catch (Exception exc) {
            throw exc
        }
    }
    return this
}

/**
 * docker registry login
 * @return
 */
def login() {
    if(this.isLoggedIn || credentialsId == ""){
        return this
    }
    // docker login
    withCredentials([usernamePassword(credentialsId: this.credentialsId, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
        def regs = this.getRegistry()
        retry(3) {
            try {
                sh "docker login ${regs} -u $USERNAME -p $PASSWORD"
            } catch (Exception exc) {
                echo "docker login err, " + exc.toString()
            }
        }
    }
    this.isLoggedIn = true;
    return this;
}

/**
 * get registry server
 * @return
 */
def getRegistry(){
    def sp = this.repo.split("/")
    if (sp.size() > 1) {
        return sp[0]
    }
    return this.repo
}
```



`Jenkinsfile`

需要先在Jenkins端创建仓库登录凭据`credential-registry`

```groovy
@Library('luffy-devops') _

pipeline {
    agent { label 'jnlp-slave'}
    options {
		timeout(time: 20, unit: 'MINUTES')
		gitLabConnection('gitlab')
	}
    environment {
        IMAGE_REPO = "172.21.51.143:5000/demo/myblog"
        IMAGE_CREDENTIAL = "credential-registry"
    }
    stages {
        stage('checkout') {
            steps {
                container('tools') {
                    checkout scm
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
    }
    post {
        success { 
            echo 'Congratulations!'
        }
        failure {
            echo 'Oh no!'
        }
    }
}
```



###### 丰富构建通知逻辑

目前的构建镜像逻辑中缺少如下内容：

- try逻辑中，若发生异常，是否该把异常抛出
  - 若直接抛出异常可能会导致多次重复的异常信息
  - 若不抛出，则如果未构建成功镜像，流水线感知不到错误
- 通知gitlab端构建任务及状态
- 构建通知格式

需要针对上述问题，做出优化

1. 优化try逻辑

   ```bash
   def build() {
       this.login()
       def isSuccess = false
       def errMsg
       retry(3) {
           try {
               sh "docker build ${this.context} -t ${this.fullAddress} -f ${this.dockerfile}"
               isSuccess = true
           }catch (Exception err) {
               //ignore
               errMsg = err.toString()
           }
           // check if build success
           if(isSuccess){
               //todo
           }else {
               // throw exception，aborted pipeline
               error errMsg
           }
           return this
       }
   }
   ```

   

2. 通知gitlab端构建任务及状态

   ```bash
   def build() {
       this.login()
       def isSuccess = false
       def errMsg = ""
       retry(3) {
           try {
               sh "docker build ${this.context} -t ${this.fullAddress} -f ${this.dockerfile} "
               isSuccess = true
           }catch (Exception err) {
               //ignore
               errMsg = err.toString()
           }
           // check if build success
           def stage = env.STAGE_NAME + '-build'
           if(isSuccess){
               updateGitlabCommitStatus(name: '${stage}', state: 'success')
           }else {
               updateGitlabCommitStatus(name: '${stage}', state: 'failed')
               // throw exception，aborted pipeline
               error errMsg
           }
   
           return this
       }
   }
   ```

   

3. 钉钉消息通知格式

   由于每个stage都需要构建通知任务，因此抽成公共的逻辑，为各stage调用

   `BuildMessage.groovy`

   ```bash
   package com.luffy.devops
   
   def updateBuildMessage(String source, String add) {
       if(!source){
           source = ""
       }
       env.BUILD_TASKS = source + add + "\n                    \n&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"
       return env.BUILD_TASKS
   }
   
   ```

   `Docker.groovy` 中调用

   ```bash
   def docker(String repo, String tag, String credentialsId, String dockerfile="Dockerfile", String context="."){
   	...
       this.msg = new BuildMessage()
       return this
   }
   
   
   ...
   
   def build() {
   ...
           // check if build success
           def stage = env.STAGE_NAME + '-build'
           if(isSuccess){
               updateGitlabCommitStatus(name: '${stage}', state: 'success')
               this.msg.updateBuildMessage(env.BUILD_TASKS, "${stage} OK...  √")
           }else {
               updateGitlabCommitStatus(name: '${stage}', state: 'failed')
               this.msg.updateBuildMessage(env.BUILD_TASKS, "${stage} Failed...  x")
               // throw exception，aborted pipeline
               error errMsg
           }
   
           return this
       }
   }
   ```



使用`Jenkinsfile`来验证上述修改是否正确：

```groovy
@Library('luffy-devops') _

pipeline {
    agent { label 'jnlp-slave'}
    options {
		timeout(time: 20, unit: 'MINUTES')
		gitLabConnection('gitlab')
	}
    environment {
        IMAGE_REPO = "172.21.51.143:5000/demo/myblog"
        IMAGE_CREDENTIAL = "credential-registry"
        DINGTALK_CREDS = credentials('dingTalk')
    }
    stages {
        stage('checkout') {
            steps {
                container('tools') {
                    checkout scm
                }
            }
        }
        stage('git-log') {
            steps {
                script{
                    sh "git log --oneline -n 1 > gitlog.file"
                    env.GIT_LOG = readFile("gitlog.file").trim()
                }
                sh 'printenv'
            }
        } 
        stage('build-image') {
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
    }
    post {
        success { 
          container('tools') {
            sh """
                curl 'https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_CREDS_PSW}' \
                    -H 'Content-Type: application/json' \
                    -d '{
                        "msgtype": "markdown",
                        "markdown": {
                            "title":"myblog",
                            "text": "😄👍 构建成功 👍😄  \n**项目名称**：luffy  \n**Git log**: ${GIT_LOG}   \n**构建分支**: ${BRANCH_NAME}   \n**构建地址**：${RUN_DISPLAY_URL}  \n**构建任务**：${env.BUILD_TASKS}"
                        }
                    }'
               """ 
            }
        }
        failure {
            echo 'Oh no!'
        }
    }
}
```



接下来需要将`push`和`login`方法做同样的改造

最终的Docker.groovy文件为：

```bash
package com.luffy.devops

/**
 *
 * @param repo
 * @param tag
 * @param credentialsId
 * @param dockerfile
 * @param context
 * @return
 */
def docker(String repo, String tag, String credentialsId, String dockerfile="Dockerfile", String context="."){
    this.repo = repo
    this.tag = tag
    this.dockerfile = dockerfile
    this.credentialsId = credentialsId
    this.context = context
    this.fullAddress = "${this.repo}:${this.tag}"
    this.isLoggedIn = false
    this.msg = new BuildMessage()
    return this
}


/**
 * build image
 * @return
 */
def build() {
    this.login()
    def isSuccess = false
    def errMsg = ""
    retry(3) {
        try {
            sh "docker build ${this.context} -t ${this.fullAddress} -f ${this.dockerfile} "
            isSuccess = true
        }catch (Exception err) {
            //ignore
            errMsg = err.toString()
        }
        // check if build success
        def stage = env.STAGE_NAME + '-build'
        if(isSuccess){
            updateGitlabCommitStatus(name: "${stage}", state: 'success')
            this.msg.updateBuildMessage(env.BUILD_TASKS, "${stage} OK...  √")
        }else {
            updateGitlabCommitStatus(name: "${stage}", state: 'failed')
            this.msg.updateBuildMessage(env.BUILD_TASKS, "${stage} Failed...  x")
            // throw exception，aborted pipeline
            error errMsg
        }

        return this
    }
}


/**
 * push image
 * @return
 */
def push() {
    this.login()
    def isSuccess = false
    def errMsg = ""
    retry(3) {
        try {
            sh "docker push ${this.fullAddress}"
            isSuccess = true
        }catch (Exception err) {
            //ignore
            errMsg = err.toString()
        }
    }
    // check if build success
    def stage = env.STAGE_NAME + '-push'
    if(isSuccess){
        updateGitlabCommitStatus(name: "${stage}", state: 'success')
        this.msg.updateBuildMessage(env.BUILD_TASKS, "${stage} OK...  √")
    }else {
        updateGitlabCommitStatus(name: "${stage}", state: 'failed')
        this.msg.updateBuildMessage(env.BUILD_TASKS, "${stage} Failed...  x")
        // throw exception，aborted pipeline
        error errMsg
    }
    return this
}

/**
 * docker registry login
 * @return
 */
def login() {
    if(this.isLoggedIn || credentialsId == ""){
        return this
    }
    // docker login
    withCredentials([usernamePassword(credentialsId: this.credentialsId, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
        def regs = this.getRegistry()
        retry(3) {
            try {
                sh "docker login ${regs} -u $USERNAME -p $PASSWORD"
            } catch (Exception ignored) {
                echo "docker login err, ${ignored.toString()}"
            }
        }
    }
    this.isLoggedIn = true;
    return this;
}

/**
 * get registry server
 * @return
 */
def getRegistry(){
    def sp = this.repo.split("/")
    if (sp.size() > 1) {
        return sp[0]
    }
    return this.repo
}
```



再次测试构建



