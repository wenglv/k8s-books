##### library集成k8s服务部署

###### library实现部署简单版

`devops.groovy`

```bash
/**
 * kubernetes deployer
 * @param resourcePath
 */
def deploy(String resourcePath){
    return new Deploy().init(resourcePath)
}

```

新增`Deploy.groovy`

```groovy
package com.luffy.devops

def init(String resourcePath){
    this.resourcePath = resourcePath
    this.msg = new BuildMessage()
    return this
}


def start(){
    try{
        //env.CURRENT_IMAGE用来存储当前构建的镜像地址，需要在Docker.groovy中设置值
        sh "sed -i 's#{{IMAGE_URL}}#${env.CURRENT_IMAGE}#g' ${this.resourcePath}/*"
        sh "kubectl apply -f ${this.resourcePath}"
        updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
        this.msg.updateBuildMessage(env.BUILD_TASKS, "${env.stage_name} OK...  √")
    } catch (Exception exc){
        updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'failed')
        this.msg.updateBuildMessage(env.BUILD_TASKS, "${env.stage_name} failed...  √")
        throw exc
    }
}
```

修改`Docker.groovy`

```groovy
def push() {
    this.login()
    def isSuccess = false
    def errMsg = ""
    retry(3) {
        try {
            sh "docker push ${this.fullAddress}"
            //把当前推送的镜像地址记录在环境变量中
            env.CURRENT_IMAGE = this.fullAddress
            isSuccess = true
        }catch (Exception err) {
            //ignore
            errMsg = err.toString()
        }

```

`Jenkinsfile` 中添加如下部分：

```groovy
        stage('deploy') {
            steps {
                container('tools') {
                    script{
                    	devops.deploy("manifests").start()
                    }
                }
            }
        }
```



###### library实现自动部署优化版

简单版本最明显的问题就是无法检测部署后的Pod状态，如果想做集成测试，通常要等到最新版本的Pod启动后再开始。因此有必要在部署的时候检测Pod是否正常运行。

比如要去检查myblog应用的pod是否部署正常，人工检查的大致步骤：

1. `kubectl -n luffy get pod`，查看pod列表

2. 找到列表中带有myblog关键字的running的pod

3. 查看上述running  pod数，是否和myblog的deployment中定义的replicas副本数一致

4. 若一致，则检查结束，若不一致，可能稍等几秒钟，再次执行相同的检查操作

5. 如果5分钟了还没有检查通过，则大概率是pod有问题，通过查看日志进一步排查



如何通过library代码实现上述过程：

1. library如何获取myblog的pod列表？

   - 首先要知道本次部署的是哪个workload，因此需要调用者传递workload的yaml文件路径

   - library解析workload.yaml文件，找到如下值：

     - pod所在的namespace
     - pod中使用的`labels`标签

   - 使用如下命令查找该workload关联的pod

     ```bash
     $ kubectl -n <namespace> get po -l <key1=value1> -l <key2=value2>
     
     # 如查找myblog的pod
     $ kubectl -n luffy get po -l app=myblog
     ```

2. 如何确定步骤1中的pod的状态？

   ```bash
   # 或者可以直接进行提取状态
   $ kubectl -n luffy get po -l app=myblog -ojsonpath='{.items[0].status.phase}'
   
   # 以json数组的形式存储
   $ kubectl -n luffy get po -l app=myblog -o json
   
   ```

3. 如何检测所有的副本数都是正常的？

   ```bash
   # 以json数组的形式存储
   $ kubectl -n luffy get po -l app=myblog -o json
   
   # 遍历数组，检测每一个pod查看是否均正常（terminating和evicted除外）
   
   ```

   

4. 如何实现在5分钟的时间内，若pod状态符合预期，则退出检测循环，若不符合预期则继续检测

   ```bash
   use( TimeCategory ) {
     def endTime = TimeCategory.plus(new Date(), TimeCategory.getMinutes(timeoutMinutes,5))
     while (true) {
       if (new Date() >= endTime) {
           //超时了，则宣告pod状态不对
           updateGitlabCommitStatus(name: 'deploy', state: 'failed')
           throw new Exception("deployment timed out...")
       }
       //循环检测当前deployment下的pod的状态
       try {
         if (this.isDeploymentReady()) {
             readyCount++
             if(readyCount > 5){
               updateGitlabCommitStatus(name: 'deploy', state: 'success')
               break;
             }
         }else {
             readyCount = 0
         }catch (Exception exc){
             echo exc.toString()
         }
         //每次检测若不满足所有pod均正常，则sleep 5秒钟后继续检测
         sleep(5)
       }
     }
   ```

   

`devops.groovy` 

通过添加参数 watch来控制是否在pipeline中观察pod的运行状态

```groovy
/**
 * 
 * @param resourcePath
 * @param watch
 * @param workloadFilePath
 * @return
 */
def deploy(String resourcePath, Boolean watch = true, String workloadFilePath){
    return new Deploy().init(resourcePath, watch, workloadFilePath)
}
```



完整版的`Deploy.groovy`

```groovy
package com.luffy.devops

import org.yaml.snakeyaml.Yaml
import groovy.json.JsonSlurperClassic
import groovy.time.TimeCategory




def init(String resourcePath, Boolean watch, String workloadFilePath) {
    this.resourcePath = resourcePath
    this.msg = new BuildMessage()
    this.watch = watch
    this.workloadFilePath = workloadFilePath
    if(!resourcePath && !workloadFilePath){
        throw Exception("illegal resource path")
    }
    return this
}


def start(){
    try{
        sh "sed -i 's#{{IMAGE_URL}}#${env.CURRENT_IMAGE}#g' ${this.resourcePath}/*"
        sh "kubectl apply -f ${this.resourcePath}"
    } catch (Exception exc){
        updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'failed')
        this.msg.updateBuildMessage(env.BUILD_TASKS, "${env.stage_name} fail...  √")
        throw exc
    }

    if (this.watch) {

        // 初始化workload文件
        initWorkload()
        String namespace = this.workloadNamespace
        String name = env.workloadName
        if(env.workloadType.toLowerCase() == "deployment"){
            echo "begin watch pod status from deployment ${env.workloadName}..."
            monitorDeployment(namespace, name)
        }else {
            //todo
            echo "workload type ${env.workloadType} does not support for now..."
        }

    }else {
        updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
        this.msg.updateBuildMessage(env.BUILD_TASKS, "${env.STAGE_NAME} OK...  √")
    }

}

def initWorkload() {
    try {
        def content = readFile this.workloadFilePath
        Yaml parser = new Yaml()
        def data = parser.load(content)
        def kind = data["kind"]
        if (!kind) {
            throw Exception("workload file ${kind} illegal, will exit pipeline!")
        }
        env.workloadType = kind
        echo "${data}"
        this.workloadNamespace = data["metadata"]["namespace"]
        if (!this.workloadNamespace){
            this.workloadNamespace = "default"
        }
        env.workloadName = data["metadata"]["name"]

    } catch (Exception exc) {
        echo "failed to readFile ${this.workloadFilePath},exception: ${exc}."
        throw exc
    }
}

/**
 *
 * @param namespace
 * @param name
 * @param timeoutMinutes
 * @param sleepTime
 * @return
 */
def monitorDeployment(String namespace, String name, int timeoutMinutes = 5, sleepTime = 3) {
    def readyCount = 0
    def readyTarget = 3
    use( TimeCategory ) {
        def endTime = TimeCategory.plus(new Date(), TimeCategory.getMinutes(timeoutMinutes))
        def lastRolling
        while (true) {
            // checking timeout
            if (new Date() >= endTime) {
                echo "timeout, printing logs..."
                this.printContainerLogs(lastRolling)
                updateGitlabCommitStatus(name: 'deploy', state: 'failed')
                this.msg.updateBuildMessage(env.BUILD_TASKS, "${env.STAGE_NAME} Failed...  x")
                throw new Exception("deployment timed out...")
            }
            // checking deployment status
            try {
                def rolling = this.getResource(namespace, name, "deployment")
                lastRolling = rolling
                if (this.isDeploymentReady(rolling)) {
                    readyCount++
                    echo "ready total count: ${readyCount}"
                    if (readyCount >= readyTarget) {
                        updateGitlabCommitStatus(name: env.STAGE_NAME, state: 'success')
                        this.msg.updateBuildMessage(env.BUILD_TASKS, "${env.STAGE_NAME} OK...  √")
                        break
                    }

                } else {
                    readyCount = 0
                    echo "reseting ready total count: ${readyCount}，print pods event logs"
                    this.printContainerLogs(lastRolling)
                    sh "kubectl get pod -n ${namespace} -o wide"
                }
            } catch (Exception exc) {
                updateGitlabCommitStatus(name: 'deploy', state: 'failed')
                this.msg.updateBuildMessage(env.BUILD_RESULT, "${env.STAGE_NAME} Failed...  ×")
                echo "error: ${exc}"
            }
            sleep(sleepTime)
        }
    }
    return this
}

def getResource(String namespace = "default", String name, String kind="deployment") {
    sh "kubectl get ${kind} -n ${namespace} ${name} -o json > ${namespace}-${name}-yaml.yml"
    def jsonStr = readFile "${namespace}-${name}-yaml.yml"
    def jsonSlurper = new JsonSlurperClassic()
    def jsonObj = jsonSlurper.parseText(jsonStr)
    return jsonObj
}


def printContainerLogs(deployJson) {
    if (deployJson == null) {
        return;
    }
    def namespace = deployJson.metadata.namespace
    def name = deployJson.metadata.name
    def labels=""
    deployJson.spec.template.metadata.labels.each { k, v ->
        labels = "${labels} -l=${k}=${v}"
    }
    sh "kubectl describe pods -n ${namespace} ${labels}"
}

def isDeploymentReady(deployJson) {
    def status = deployJson.status
    def replicas = status.replicas
    def unavailable = status['unavailableReplicas']
    def ready = status['readyReplicas']
    if (unavailable != null) {
        return false
    }
    def deployReady = (ready != null && ready == replicas)
    // get pod information
    if (deployJson.spec.template.metadata != null && deployReady) {
        if (deployJson.spec.template.metadata.labels != null) {
            def labels=""
            def namespace = deployJson.metadata.namespace
            def name = deployJson.metadata.name
            deployJson.spec.template.metadata.labels.each { k, v ->
                labels = "${labels} -l=${k}=${v}"
            }
            if (labels != "") {
                sh "kubectl get pods -n ${namespace} ${labels} -o json > ${namespace}-${name}-json.json"
                def jsonStr = readFile "${namespace}-${name}-json.json"
                def jsonSlurper = new JsonSlurperClassic()
                def jsonObj = jsonSlurper.parseText(jsonStr)
                def totalCount = 0
                def readyCount = 0
                jsonObj.items.each { k, v ->
                    echo "pod phase ${k.status.phase}"
                    if (k.status.phase != "Terminating" && k.status.phase != "Evicted") {
                        totalCount++;
                        if (k.status.phase == "Running") {
                            readyCount++;
                        }
                    }
                }
                echo "Pod running count ${totalCount} == ${readyCount}"
                return totalCount > 0 && totalCount == readyCount
            }
        }
    }
    return deployReady
}

```



修改`Jenkinsfile` 调用部分：

```bash
        stage('deploy') {
            steps {
                container('tools') {
                    script{
                    	devops.deploy("manifests", true, "manifests/deployment.yaml").start()
                    }
                }
            }
        }
```



