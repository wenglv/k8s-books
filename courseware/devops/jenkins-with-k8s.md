#### 工具集成与Jenkinsfile实践篇

1. Jenkins如何对接kubernetes集群
2. 使用kubernetes的Pod-Template来作为动态的agent执行Jenkins任务
3. 如何制作agent容器实现不同类型的业务的集成
4. 集成代码扫描、docker镜像自动构建、k8s服务部署、自动化测试

##### 集成Kubernetes

###### 插件安装及配置

[插件官方文档](https://plugins.jenkins.io/kubernetes/)

1. [系统管理] -> [插件管理] -> [搜索kubernetes]->直接安装

   若安装失败，请先更新[ bouncycastle API Plugin](https://plugins.jenkins.io/bouncycastle-api)并重新启动Jenkins

2. [系统管理] -> [系统配置] -> [Add a new cloud]

3. 配置地址信息

   -  Kubernetes 地址: https://kubernetes.default
   -  Kubernetes 命名空间：jenkins
   -  服务证书不用写（我们在安装Jenkins的时候已经指定过serviceAccount），均使用默认
   -  连接测试，成功会提示：Connection test successful
   -  Jenkins地址：http://jenkins:8080
   -  Jenkins 通道 ：jenkins:50000

4. 配置Pod Template

   - 名称：jnlp-slave

   - 命名空间：jenkins

   - 标签列表：jnlp-slave，作为agent的label选择用

   - 连接 Jenkins 的超时时间（秒） ：300，设置连接jenkins超时时间

   - 工作空间卷：选择hostpath，设置/opt/jenkins,注意需要设置目录权限，否则Pod没有权限
    ![](images\workspace-volume.png)
     ```bash
     $ chown -R 1000:1000 /opt/jenkins
     $ chmod 777 /opt/jenkins
     ```

     

###### 演示动态slave pod

```bash
# 为准备运行jnlp-slave-agent的pod的节点打上label
$ kubectl label node k8s-slave1 agent=true

### 回放一次多分支流水线develop分支
agent { label 'jnlp-slave'}

```

执行任务，会下载默认的jnlp-slave镜像，地址为jenkins/inbound-agent:4.11-1-jdk11，我们可以先在k8s-master节点拉取下来该镜像：

```bash
$ docker pull jenkins/inbound-agent:4.11-1-jdk11

```

保存jenkinsfile提交后，会出现报错，因为我们的agent已经不再是宿主机，而是Pod中的容器内，报错如下：

![](images\gitlab-no-docker-err.png)

因此我们需要将用到的命令行工具集成到Pod的容器内，但是思考如下问题：

- 目前是用的jnlp的容器，是java的环境，我们在此基础上需要集成很多工具，能不能创建一个新的容器，让新容器来做具体的任务，jnlp-slave容器只用来负责连接jenkins-master
- 针对不同的构建环境（java、python、go、nodejs），可以制作不同的容器，来执行对应的任务

> 

###### Pod-Template中容器镜像的制作

为解决上述问题，我们制作一个tools镜像，集成常用的工具，来完成常见的构建任务，需要注意的几点：

- 使用alpine基础镜像，自身体积比较小
- 替换国内安装源
- 为了使用docker，安装了docker
- 为了克隆代码，安装git
- 为了后续做python的测试等任务，安装python环境
- 为了在容器中调用kubectl的命令，拷贝了kubectl的二进制文件
- 为了认证kubectl，需要在容器内部生成.kube目录及config文件

```bash
$ mkdir tools;
$ cd tools;
$ cp `which kubectl` .
$ cp ~/.kube/config .

```

*Dockerfile*

`jenkins/custom-images/tools/Dockerfile`

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

```

执行镜像构建并推送到仓库中：

```bash
$ docker build . -t 172.21.51.143:5000/devops/tools:v1
$ docker push 172.21.51.143:5000/devops/tools:v1

```

我们可以直接使用该镜像做测试：

```bash
## 启动临时镜像做测试
$ docker run --rm -ti 172.21.51.143:5000/devops/tools:v1 bash
# / git clone http://xxxxxx.git
# / kubectl get no
# / python3
#/ docker

## 重新挂载docker的sock文件
docker run -v /var/run/docker.sock:/var/run/docker.sock --rm -ti 172.21.51.143:5000/devops/tools:v1 bash

```



###### 实践通过Jenkinsfile实现demo项目自动发布到kubenetes环境

更新Jenkins中的PodTemplate，添加tools镜像，注意同时要先添加名为jnlp的container，因为我们是使用自定义的PodTemplate覆盖掉默认的模板：

![](images\pod-template-jnlp.png)

在卷栏目，添加卷，Host Path Volume，不然在容器中使用docker会提示docker服务未启动

![](images\jenkins-docker-sock.png)



tools容器做好后，我们需要对Jenkinsfile做如下调整：

`jenkins/pipelines/p8.yaml`

```bash
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
        stage('printenv') {
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

> 