##### 集成robot自动化测试

关于集成测试，我们需要知道的几点:

- 测试人员进行编写
- 侧重于不同模块的接口调用，对新加的功能进行验证
- 注重新版本对以前的集成用例进行回归

因此，更多的应该是跨模块去测试，而且测试用例是测试人员去维护，因此不适合把代码放在开发的git仓库中。

本节要实现的工作：

1. 创建新的git仓库`robot-cases`，用于存放robot测试用例
2. 为`robot-cases`项目创建Jenkinsfile
3. 配置Jenkins任务，实现该项目的自动化执行
4. 在myblog模块的流水线中，对该流水线项目进行调用



###### 初始化robot-cases项目

1. 新建gitlab项目，名称为`robot-cases`

2. clone到本地

3. 本地拷贝myblog项目的`robot.txt`

   ```bash
   robot-cases/
   └── myblog
       └── robot.txt
   ```

###### 配置Jenkinsfile及自动化任务

```bash
robot-cases/
├── Jenkinsfile
└── myblog
    └── robot.txt
```

`Jenkinsfile`

多个业务项目的测试用例都在一个仓库中，因此需要根据参数设置来决定执行哪个项目的用例

```groovy
pipeline {
    agent {
        label 'jnlp-slave'
    }

	options {
        timeout(time: 20, unit: 'MINUTES')
		gitLabConnection('gitlab')
	}
    stages {
        stage('checkout') {
            steps {
                container('tools') {
                    checkout scm
                }
            }
        }
        stage('Test') {
            steps {
                script {
                    container('tools'){
                        switch(env.comp){
                            case "myblog":
                                env.testDir = "myblog"
                                break
                            case "business1":
                                env.testDir = "business1"
                                break
                            default:
                                env.testDir = "all"
                                break
                        }
                        sh 'robot -d artifacts/ ${testDir}/*'
                        step([
                            $class : 'RobotPublisher',
                            outputPath: 'artifacts/',
                            outputFileName : "output.xml",
                            disableArchiveOutput : false,
                            passThreshold : 100,
                            unstableThreshold: 80.0,
                            onlyCritical : true,
                            otherFiles : "*.png"
                        ])
                        archiveArtifacts artifacts: 'artifacts/*', fingerprint: true
                    }
                }
            }
        }
    }
}
```



如何实现将`env.comp` 传递进去？

配置流水线的参数化构建任务并验证参数化构建



###### library集成触发任务

由于多个项目均需要触发自动构建，因此可以在library中抽象方法，实现接收comp参数，并在library中实现对`robot-cases`项目的触发。

![](images\robot-trigger.png)

`devops.groovy`

```groovy
/**
 * 
 * @param comp
 * @return
 */
def robotTest(String comp=""){
    new Robot().acceptanceTest(comp)
}
```



新建`Robot.groovy`文件

```groovy
package com.luffy.devops

def acceptanceTest(comp) {
    try{
        echo "Trigger to execute Acceptance Testing"
        def rf = build job: 'robot-cases',
                parameters: [
                        string(name: 'comp', value: comp)
                ],
                wait: true,
                propagate: false
        def result = rf.getResult()
        def msg = "${env.STAGE_NAME}... "
        if (result == "SUCCESS"){
            msg += "√ success"
        }else if(result == "UNSTABLE"){
            msg += "⚠ unstable"
        }else{
            msg += "× failure"
        }
        echo rf.getAbsoluteUrl()
        env.ROBOT_TEST_URL = rf.getAbsoluteUrl()
        new BuildMessage().updateBuildMessage(env.BUILD_TASKS, msg)
    } catch (Exception exc) {
        echo "trigger  execute Acceptance Testing exception: ${exc}"
        new BuildMessage().updateBuildMessage(env.BUILD_RESULT, msg)
    }
}

```



修改`Jenkinsfile`测试调用

```groovy
        stage('integration test') {
            steps {
                container('tools') {
                    script{
                    	devops.robotTest("myblog")
                    }
                }
            }
        }
```

