##### Master-Slaves（agent）模式

上面演示的任务，默认都是在master节点执行的，多个任务都在master节点执行，对master节点的性能会造成一定影响，如何将任务分散到不同的节点，做成多slave的方式？

1. 添加slave节点

   - 系统管理 -> 节点管理 -> 新建节点
   - 比如添加172.21.51.68，选择固定节点，保存
   - 远程工作目录/opt/jenkins_jobs
   - 标签为任务选择节点的依据，如172.21.51.68
   - 启动方式选择通过java web启动代理，代理是运行jar包，通过JNLP（是一种允许客户端启动托管在远程Web服务器上的应用程序的协议 ）启动连接到master节点服务中

   ![](images\jenkins-new-node.jpg)

2. 执行java命令启动agent服务

   ```bash
   ## 登录172.21.51.68，下载agent.jar
   $ wget http://jenkins.luffy.com/jnlpJars/agent.jar
   ## 会提示找不到agent错误，因为没有配置地址解析，由于连接jenkins master会通过50000端口，直接使用cluster-ip
   $ kubectl -n jenkins get svc #在master节点执行查询cluster-ip地址
   NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)              AGE
   jenkins   ClusterIP   10.99.204.208   <none>        8080/TCP,50000/TCP   4h8m
   
   ## 再次回到68节点
   $ wget 10.99.204.208:8080/jnlpJars/agent.jar
   $ java -jar agent.jar -jnlpUrl http://10.99.204.208:8080/computer/172.21.51.68/slave-agent.jnlp -secret 4be4d164f861d2830835653567867a1e695b30c320d35eca2be9f5624f8712c8 -workDir "/opt/jenkins_jobs"
   ...
   INFO: Remoting server accepts the following protocols: [JNLP4-connect, Ping]
   Apr 01, 2020 7:03:51 PM hudson.remoting.jnlp.Main$CuiListener status
   INFO: Agent discovery successful
     Agent address: 10.99.204.208
     Agent port:    50000
     Identity:      e4:46:3a:de:86:24:8e:15:09:13:3d:a7:4e:07:04:37
   Apr 01, 2020 7:03:51 PM hudson.remoting.jnlp.Main$CuiListener status
   INFO: Handshaking
   Apr 01, 2020 7:03:51 PM hudson.remoting.jnlp.Main$CuiListener status
   INFO: Connecting to 10.99.204.208:50000
   Apr 01, 2020 7:03:51 PM hudson.remoting.jnlp.Main$CuiListener status
   INFO: Trying protocol: JNLP4-connect
   Apr 01, 2020 7:04:02 PM hudson.remoting.jnlp.Main$CuiListener status
   INFO: Remote identity confirmed: e4:46:3a:de:86:24:8e:15:09:13:3d:a7:4e:07:04:37
   Apr 01, 2020 7:04:03 PM hudson.remoting.jnlp.Main$CuiListener status
   INFO: Connected
   ```

   若出现如下错误:

   ```bash
   SEVERE: http://jenkins.luffy.com/ provided port:50000 is not reachable
   java.io.IOException: http://jenkins.luffy.com/ provided port:50000 is not reachable
           at org.jenkinsci.remoting.engine.JnlpAgentEndpointResolver.resolve(JnlpAgentEndpointResolver.java:311)
           at hudson.remoting.Engine.innerRun(Engine.java:689)
           at hudson.remoting.Engine.run(Engine.java:514)
   ```

   可以选择： 配置从节点 -> 高级 -> Tunnel连接位置，参考下图进行设置:

   ![](images\slave-tunnel.jpg)

3. 查看Jenkins节点列表，新节点已经处于可用状态

   ![](images\jenkins-node-lists.jpg)

4. 测试使用新节点执行任务

   - 配置free项目

   - 限制项目的运行节点 ，标签表达式选择172.21.51.68

   - 立即构建

   - 查看构建日志

     ```bash
     Started by user admin
     Running as SYSTEM
     Building remotely on 172.21.51.68 in workspace /opt/jenkins_jobs/workspace/free-demo
     using credential gitlab-user
     Cloning the remote Git repository
     Cloning repository http://gitlab.luffy.com/root/myblog.git
      > git init /opt/jenkins_jobs/workspace/free-demo # timeout=10
      ...
     
     ```

> 