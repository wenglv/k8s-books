##### library集成代码扫描

sonarqube代码扫描作为通用功能，同样可以使用library实现。

`devops.groovy`

```groovy
/**
 * sonarqube scanner
 * @param projectVersion
 * @param waitScan
 * @return
 */
def scan(String projectVersion="", Boolean waitScan = true) {
    return new Sonar().init(projectVersion, waitScan)
}
```

新建`Sonar.groovy`

- 可以传递projectVersion作为sonarqube的扫描版本
- 参数waitScan来设置是否等待本次扫描是否通过

```groovy
package com.luffy.devops


def init(String projectVersion="", Boolean waitScan = true) {
    this.waitScan = waitScan
    this.msg = new BuildMessage()
    if (projectVersion == ""){
        projectVersion = sh(returnStdout: true, script: 'git log --oneline -n 1|cut -d " " -f 1')
    }
    sh "echo '\nsonar.projectVersion=${projectVersion}' >> sonar-project.properties"
    sh "cat sonar-project.properties"
    return this
}

def start() {
    try {
        this.startToSonar()
    }
    catch (Exception exc) {
        throw exc
    }
    return this
}

def startToSonar() {
    withSonarQubeEnv('sonarqube') {
        sh "sonar-scanner -X;"
        sleep 5
    }
    if(this.waitScan){
        //wait 3min
        timeout(time: 3, unit: 'MINUTES') {
            def qg = waitForQualityGate()
            String stage = "${env.stage_name}"
            if (qg.status != 'OK') {
                this.msg.updateBuildMessage(env.BUILD_TASKS, "${stage} Failed...  ×")
                updateGitlabCommitStatus(name: "${stage}", state: 'failed')
                error "Pipeline aborted due to quality gate failure: ${qg.status}"
            }else{
                this.msg.updateBuildMessage(env.BUILD_RESULT, "${stage} OK...  √")
                updateGitlabCommitStatus(name: "${stage}", state: 'success')
            }
        }
    }else{
        echo "skip waitScan"
    }
    return this
}
```



`Jenkinsfile`新增如下部分：

```groovy
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
```

