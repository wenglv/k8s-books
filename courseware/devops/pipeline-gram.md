#### 流水线入门

![](images\pipeline-factory.jpeg)

[官方文档](https://jenkins.io/zh/doc/book/pipeline/getting-started/)

![](images\realworld-pipeline-flow.png)

为什么叫做流水线，和工厂产品的生产线类似，pipeline是从源码到发布到线上环境。关于流水线，需要知道的几个点：

- 重要的功能插件，帮助Jenkins定义了一套工作流框架；

- Pipeline 的实现方式是一套 Groovy DSL（ 领域专用语言 ），所有的发布流程都可以表述为一段 Groovy 脚本；

- 将WebUI上需要定义的任务，以脚本代码的方式表述出来；

- 帮助jenkins实现持续集成CI（Continue Integration）和持续部署CD（Continue Deliver）的重要手段；

##### 流水线基础语法

[官方文档](https://jenkins.io/zh/doc/book/pipeline/syntax/)

两种语法类型：

- Scripted Pipeline，脚本式流水线，最初支持的类型
- Declarative Pipeline，声明式流水线，为Pipeline plugin在2.5版本之后新增的一种脚本类型，后续Open Blue Ocean所支持的类型。与原先的Scripted Pipeline一样，都可以用来编写脚本。Declarative Pipeline 是后续Open Blue Ocean所支持的类型，写法简单，支持内嵌Scripted Pipeline代码

*为与BlueOcean脚本编辑器兼容，通常建议使用Declarative Pipeline的方式进行编写,从jenkins社区的动向来看，很明显这种语法结构也会是未来的趋势。*

###### 脚本示例

```json
pipeline { 
    agent {label '172.21.51.68'}
    environment { 
        PROJECT = 'myblog'
    }
    stages {
        stage('Checkout') { 
            steps { 
                checkout scm 
            }
        }
        stage('Build') { 
            steps { 
                sh 'make' 
            }
        }
        stage('Test'){
            steps {
                sh 'make check'
                junit 'reports/**/*.xml' 
            }
        }
        stage('Deploy') {
            steps {
                sh 'make publish'
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

###### 脚本解释：

- `checkout`步骤为检出代码; `scm`是一个特殊变量，指示`checkout`步骤克隆触发此Pipeline运行的特定修订 

- agent：指明使用哪个agent节点来执行任务，定义于pipeline顶层或者stage内部

  - any，可以使用任意可用的agent来执行

  - label，在提供了标签的 Jenkins 环境中可用的代理上执行流水线或阶段。 例如: `agent { label 'my-defined-label' }`，最常见的使用方式

  - none，当在 `pipeline` 块的顶部没有全局代理， 该参数将会被分配到整个流水线的运行中并且每个 `stage` 部分都需要包含他自己的 `agent` 部分。比如: `agent none`

  - docker， 使用给定的容器执行流水线或阶段。 在指定的节点中，通过运行容器来执行任务

    ```json
    agent {
        docker {
            image 'maven:3-alpine'
            label 'my-defined-label'
            args  '-v /tmp:/tmp'
        }
    }
    
    ```

- options:  允许从流水线内部配置特定于流水线的选项。  

  -  buildDiscarder , 为最近的流水线运行的特定数量保存组件和控制台输出。例如: `options { buildDiscarder(logRotator(numToKeepStr: '10')) }` 
  -  disableConcurrentBuilds ,不允许同时执行流水线。 可被用来防止同时访问共享资源等。 例如: `options { disableConcurrentBuilds() }`
  -  timeout ,设置流水线运行的超时时间, 在此之后，Jenkins将中止流水线。例如: `options { timeout(time: 1, unit: 'HOURS') }`
  -  retry，在失败时, 重新尝试整个流水线的指定次数。 For example: `options { retry(3) }`

- environment:  指令制定一个 键-值对序列，该序列将被定义为所有步骤的环境变量 

- stages: 包含一系列一个或多个 [stage](https://jenkins.io/zh/doc/book/pipeline/syntax/#stage)指令, `stages` 部分是流水线描述的大部分"work" 的位置。 建议 `stages` 至少包含一个 [stage](https://jenkins.io/zh/doc/book/pipeline/syntax/#stage) 指令用于连续交付过程的每个离散部分,比如构建, 测试, 和部署。 

  ```groovy
  pipeline {
      agent any
      stages { 
          stage('Example') {
              steps {
                  echo 'Hello World'
              }
          }
      }
  }
  
  ```

- steps: 在给定的 `stage` 指令中执行的定义了一系列的一个或多个[steps](https://jenkins.io/zh/doc/book/pipeline/syntax/#declarative-steps)。

- post:  定义一个或多个[steps](https://jenkins.io/zh/doc/book/pipeline/syntax/#declarative-steps) ，这些阶段根据流水线或阶段的完成情况而运行`post` 支持以下 [post-condition](https://jenkins.io/zh/doc/book/pipeline/syntax/#post-conditions) 块中的其中之一: `always`, `changed`, `failure`, `success`, `unstable`, 和 `aborted`。

  -  always, 无论流水线或阶段的完成状态如何，都允许在 `post` 部分运行该步骤
  -  changed, 当前流水线或阶段的完成状态与它之前的运行不同时，才允许在 `post` 部分运行该步骤
  -  failure,  当前流水线或阶段的完成状态为"failure"，才允许在 `post` 部分运行该步骤, 通常web UI是红色
  -  success,  当前流水线或阶段的完成状态为"success"，才允许在 `post` 部分运行该步骤, 通常web UI是蓝色或绿色
  -  unstable,  当前流水线或阶段的完成状态为"unstable"，才允许在 `post` 部分运行该步骤, 通常由于测试失败,代码违规等造成。通常web UI是黄色 
  -  aborted， 只有当前流水线或阶段的完成状态为"aborted"，才允许在 `post` 部分运行该步骤, 通常由于流水线被手动的aborted。通常web UI是灰色 

创建pipeline示意：

新建任务 -> 流水线 

`jenkins/pipelines/p1.yaml`

```bash
pipeline {
   agent {label '172.21.51.68'}
   environment { 
      PROJECT = 'myblog'
   }
   stages {
      stage('printenv') {
         steps {
            echo 'Hello World'
            sh 'printenv'
         }
      }
      stage('check') {
         steps {
            checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'gitlab-user', url: 'http://gitlab.luffy.com/root/myblog.git']]])
         }
      }
      stage('build-image') {
         steps {
            sh 'docker build . -t myblog:latest -f Dockerfile'
         }
      }
      stage('send-msg') {
         steps {
            sh """
            curl 'https://oapi.dingtalk.com/robot/send?access_token=4778abd23dbdbaf66fc6f413e6ab9c0103a039b0054201344a22a5692cdcc54e' \
   -H 'Content-Type: application/json' \
   -d '{"msgtype": "text", 
        "text": {
             "content": "我就是我, 是不一样的烟火"
        }
      }'
      """
         }
      }
   }
}

```

点击“立即构建”，同样的，我们可以配置触发器，使用webhook的方式接收项目的push事件，

- 构建触发器选择 Build when a change is pushed to GitLab. 
- 生成 Secret token 
- 配置gitlab，创建webhook，发送test push events测试

###### Blue Ocean:

[官方文档]( https://jenkins.io/zh/doc/book/blueocean/getting-started/ )

我们需要知道的几点：

- 是一个插件， 旨在为Pipeline提供丰富的体验 ；
- 连续交付（CD）Pipeline的复杂可视化，允许快速和直观地了解Pipeline的状态；
- 目前支持的类型仅针对于Pipeline，尚不能替代Jenkins 经典版UI

思考：

1. 每个项目都把大量的pipeline脚本写在Jenkins端，对于谁去维护及维护成本是一个问题
2. 没法做版本控制