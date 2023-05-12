##### library与Jenkins集成

先来看一下如何使用shared library实现最简单的helloworld输出功能，来理清楚使用shared library的流程。

###### Hello.groovy

```groovy
package com.luffy.devops

/**
* @author Yongxin
* @version v0.1
 */

/**
 * say hello
 * @param content
 */
def init(String content) {
    this.content = content
    return this
}


def sayHi() {
    echo "Hi, ${this.content},how are you?"
    return this
}

def answer() {
    echo "${this.content}: fine, thank you, and you?"
    return this
}

def sayBye() {
    echo "i am fine too , ${this.content}, Bye!"
    return this
}
```

`.gitignore`

```
.idea/*
.vscode/*
out
```

```bash
# git记住密码
$ git config --global credential.helper store
```



在gitlab创建项目，把library代码推送到镜像仓库。





###### 配置Jenkins

[系统管理] -> [系统设置] -> [ **Global Pipeline Libraries** ]

- Library Name：luffy-devops
- Default Version：master
- Source Code Management：Git



###### Jenkinsfile中引用

`jenkins/pipelines/p11.yaml`

```groovy
@Library('luffy-devops') _

pipeline {
    agent { label 'jnlp-slave'}

    stages {
        stage('hello-devops') {
            steps {
                script {
                    devops.hello("树哥").sayHi().answer().sayBye()
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
        always { 
            echo 'I will always say Hello again!'
        }
    }
}
```



创建`vars/devops.groovy`

```groovy
import com.luffy.devops.Hello

def hello(String content) {
    return new Hello().init(content)
}
```



