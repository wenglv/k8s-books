##### 小结

Jenkins-shared-library的代码地址： https://gitee.com/agagin/jenkins-shared-library 

目标：让devops流程更好用

- 项目更简便的接入
- devops流程更方便维护

思路：把各项目中公用的逻辑，抽象成方法，放到独立的library项目中，在各项目中引入shared-library项目，调用library提供的方法。

- 镜像构建、推送
- k8s服务部署、监控
- 钉钉消息推送
- 代码扫描
- robot集成测试

为了兼容多环境的CICD，因此采用模板与数据分离的方式，项目中的定义模板，shared-library中实现模板替换。为了实现shared-library与各项目解耦，使用configmap来维护模板与真实数据的值，思路是约定大于配置。