##### library实现即时消息推送

###### 实现消息通知

由于发送消息通知属于通用的功能，因此有必要把消息通知抽象成为通用的功能。

`devops.groovy`

```bash
/**
 * notificationSuccess
 * @param project
 * @param receiver
 * @param credentialsId
 * @param title
 * @return
 */
def notificationSuccess(String project, String receiver="dingTalk", String credentialsId="dingTalk", String title=""){
    new Notification().getObject(project, receiver, credentialsId, title).notification("success")
}

/**
 * notificationFailed
 * @param project
 * @param receiver
 * @param credentialsId
 * @param title
 * @return
 */
def notificationFailed(String project, String receiver="dingTalk", String credentialsId="dingTalk", String title=""){
    new Notification().getObject(project, receiver, credentialsId, title).notification("failure")
}
```

新建`Notification.groovy`文件：

```groovy
package com.luffy.devops

/**
 *
 * @param type
 * @param credentialsId
 * @param title
 * @return
 */
def getObject(String project, String receiver, String credentialsId, String title) {
    this.project = project
    this.receiver = receiver
    this.credentialsId = credentialsId
    this.title = title
    return this
}


def notification(String type){
    String msg ="😄👍 ${this.title} 👍😄"

    if (this.title == "") {
        msg = "😄👍 流水线成功啦 👍😄"
    }
    // failed
    if (type == "failure") {
        msg ="😖❌ ${this.title} ❌😖"
        if (this.title == "") {
            msg = "😖❌ 流水线失败了 ❌😖"
        }
    }
	String title = msg
    // rich notify msg
    msg = genNotificationMessage(msg)
    if( this.receiver == "dingTalk") {
        try {
            new DingTalk().markDown(title, msg, this.credentialsId)
        } catch (Exception ignored) {}
    }else if(this.receiver == "wechat") {
        //todo
    }else if (this.receiver == "email"){
        //todo
    }else{
        error "no support notify type!"
    }
}


/**
 * get notification msg
 * @param msg
 * @return
 */
def genNotificationMessage(msg) {
    // project
    msg = "${msg}  \n  **项目名称**: ${this.project}"
    // get git log
    def gitlog = ""
    try {
        sh "git log --oneline -n 1 > gitlog.file"
        gitlog = readFile "gitlog.file"
    } catch (Exception ignored) {}

    if (gitlog != null && gitlog != "") {
        msg = "${msg}  \n  **Git log**: ${gitlog}"
    }
    // get git branch
    def gitbranch = env.BRANCH_NAME
    if (gitbranch != null && gitbranch != "") {
        msg = "${msg}  \n  **Git branch**: ${gitbranch}"
    }
    // build tasks
    msg = "${msg}  \n  **Build Tasks**: ${env.BUILD_TASKS}"

    // get buttons
    msg = msg + getButtonMsg()
    return msg
}
def getButtonMsg(){
    String res = ""
    def  buttons = [
            [
                    "title": "查看流水线",
                    "actionURL": "${env.RUN_DISPLAY_URL}"
            ],
            [
                    "title": "代码扫描结果",
                    "actionURL": "http://sonar.luffy.com/dashboard?id=${this.project}"
            ]
    ]
    buttons.each() {
        if(res == ""){
            res = "   \n >"
        }
        res = "${res} --- ["+it["title"]+"]("+it["actionURL"]+") "
    }
    return res
}
```



新建`DingTalk.groovy`文件：

```groovy
package com.luffy.devops

import groovy.json.JsonOutput


def sendRequest(method, data, credentialsId, Boolean verbose=false, codes="100:399") {
    def reqBody = new JsonOutput().toJson(data)
    withCredentials([usernamePassword(credentialsId: credentialsId, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
        def response = httpRequest(
                httpMode:method,
                url: "https://oapi.dingtalk.com/robot/send?access_token=${PASSWORD}",
                requestBody:reqBody,
                validResponseCodes: codes,
                contentType: "APPLICATION_JSON",
                quiet: !verbose
        )
    }
}

def markDown(String title, String text, String credentialsId, Boolean verbose=false) {
    def data = [
            "msgtype": "markdown",
            "markdown": [
                    "title": title,
                    "text": text
            ]
    ]
    this.sendRequest("POST", data, credentialsId, verbose)
}
```



需要用到Http Request来发送消息，安装一下插件：http_request



###### jenkinsfile调用

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
                devops.notificationSuccess("myblog","dingTalk")
            }
        }
        failure {
            script{
                devops.notificationFailed("myblog","dingTalk")
            }
        }
    }
}
```

