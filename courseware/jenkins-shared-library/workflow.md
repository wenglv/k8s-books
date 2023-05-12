### 基于sharedLibrary进行CI/CD流程的优化

由于公司内部项目众多，大量的项目使用同一套流程做CICD

- 那么势必会存在大量的重复代码
- 一旦某个公共的地方需要做调整，每个项目都需要修改

因此本章主要通过使用groovy实现Jenkins的sharedLibrary的开发，以提取项目在CICD实践过程中的公共逻辑，提供一系列的流程的接口供公司内各项目调用。

开发完成后，对项目进行Jenkinsfile的改造，最后仅需通过简单的Jenkinsfile的配置，即可优雅的完成CICD流程的整个过程，此方式已在大型企业内部落地应用。



##### Library工作模式

由于流水线被组织中越来越多的项目所采用，常见的模式很可能会出现。 在多个项目之间共享流水线有助于减少冗余并保持代码 "DRY"。

流水线支持引用 "共享库" ，可以在外部源代码控制仓库中定义并加载到现有的流水线中。

```
@Library('my-shared-library') _
```

在实际运行过程中，会把library中定义的groovy功能添加到构建目录中：

```bash
/var/jenkins_home/jobs/test-maven-build/branches/feature-CDN-2904.cm507o/builds/2/libs/my-shared-library/vars/devops.groovy
```

使用library后，Jenkinsfile大致的样子如下：

```bash
@Library('my-shared-library') _

...
  stages {
    stage('build image') {
      steps {
         container('tools') {
           devops.buildImage("Dockerfile","172.21.51.143:5000/demo:latest")
         }
      }
    }
  }
  
  post {
    success {
      script {
          container('tools') {
              devops.notificationSuccess("dingTalk")
          }
      }
    }
  }
...
```

