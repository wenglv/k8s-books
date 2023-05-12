##### Jenkins定制化容器

由于每次新部署Jenkins环境，均需要安装很多必要的插件，因此考虑把插件提前做到镜像中

*Dockerfile*

```dockerfile
FROM jenkinsci/blueocean:1.25.2
LABEL maintainer="inspur_lyx@hotmail.com"

ENV JENKINS_UC https://updates.jenkins-zh.cn
ENV JENKINS_UC_DOWNLOAD https://mirrors.tuna.tsinghua.edu.cn/jenkins
ENV JENKINS_OPTS="-Dhudson.model.UpdateCenter.updateCenterUrl=https://updates.jenkins-zh.cn/update-center.json"
ENV JENKINS_OPTS="-Djenkins.install.runSetupWizard=false"

## 用最新的插件列表文件替换默认插件文件
COPY plugins.txt /usr/share/jenkins/ref/

## 执行插件安装
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

```

*plugins.txt*

```python
ace-editor:1.1
allure-jenkins-plugin:2.28.1
ant:1.10
antisamy-markup-formatter:1.6
apache-httpcomponents-client-4-api:4.5.10-1.0
authentication-tokens:1.3
...

```

*get_plugin.sh*

> admin:123456@localhost 需要替换成Jenkins的用户名、密码及访问地址

```bash
#!/usr/bin/env bash
curl -sSL  "http://admin:admin@jenkins.luffy.com/pluginManager/api/xml?depth=1&xpath=/*/*/shortName|/*/*/version&wrapper=plugins" | perl -pe 's/.*?<shortName>([\w-]+).*?<version>([^<]+)()(<\/\w+>)+/\1:\2\n/g'|sed 's/ /:/' > plugins.txt

```

```bash
## 执行构建，定制jenkins容器
$ docker build . -t 172.21.51.143:5000/jenkins:v20200414 -f Dockerfile
$ docker push 172.21.51.143:5000/jenkins:v20200414

```

至此，我们可以使用定制化的镜像启动jenkins服务

```bash
## 删掉当前服务
$ kubectl delete -f jenkins-all.yaml

## 删掉已挂载的数据
$ rm -rf /var/jenkins_home

## 替换使用定制化镜像
$ sed -i 's#jenkinsci/blueocean#172.21.51.143:5000/jenkins:v20200404#g' jenkins-all.yaml

## 重新创建服务
$ kubectl create -f jenkins-all.yaml

```



##### 本章小结

自由风格项目弊端：

- 任务的完成需要在Jenkins端维护大量的配置
- 没法做版本控制
- 可读性、可移植性很差，不够优雅

