#### Spring Boot交付实践

##### 从零开始创建Spring Boot项目

 通过File > New > Project，新建工程，选择Spring Initializr

![](images\new-springboot-1.jpg)

配置Project Metadata：

![](images\new-springboot-2.jpg)

配置Dependencies依赖包：

选择：Web分类中的Spring web和Template Engines中的Thymeleaf

![](images\new-springboot-3.jpg)

配置maven settings.xml：

默认使用IDE自带的maven，换成自己下载的，下载地址：

链接: https://pan.baidu.com/s/1z9dRGv_4bS1uxBtk5jsZ2Q 提取码: 3gva

解压后放到`D:\software\apache-maven-3.6.3`,修改`D:\software\apache-maven-3.6.3\conf\settings.xml` 文件：

```bash
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <localRepository>D:\opt\maven-repo</localRepository>
  
  <pluginGroups>
  </pluginGroups>

  <proxies>
  </proxies>

  <servers>
  </servers>

  <mirrors>
	    <mirror>
            <id>alimaven</id>
            <mirrorOf>central</mirrorOf>
            <name>aliyun maven</name>
            <url>http://maven.aliyun.com/nexus/content/repositories/central/</url>
        </mirror>
        <mirror>
            <id>nexus-aliyun</id>
            <mirrorOf>*</mirrorOf>
            <name>Nexus aliyun</name>
            <url>http://maven.aliyun.com/nexus/content/groups/public</url>
        </mirror>
  </mirrors>

</settings>
```



![](images\new-springboot-4.png)

![](images\new-springboot-5.jpg)

> springboot版本为2.3.6.RELEASE

默认生成的SpringBoot版本为2.5.2，换成2.3.6.RELEASE版本

```xml
...
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
...
```



直接启动项目并访问本地服务：`localhost:8080`

##### 编写功能代码

创建controller包及`HelloController.java`文件

```java
package com.luffy.springbootdemo.controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    public String hello(String name) {
        return "Hello, " + name;
    }
}
```

保存并在浏览器中访问`localhost:8080/hello?name=luffy`

如果页面复杂，如何实现？

在`resources/templates/`目录下新建`index.html`

```html
<!DOCTYPE html>
<html>
<head>
    <title>Devops</title>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
</head>
<body>
<div class="container">
    <h3 th:text="${requestname}"></h3>
    <a id="rightaway" href="#" th:href="@{/rightaway}" >立即返回</a>
    <a id="sleep" href="#" th:href="@{/sleep}">延时返回</a>
</div>
</body>
</html>
```

完善`HelloController.java`的内容：

```java
package com.luffy.springbootdemo.controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.servlet.ModelAndView;

@RestController
public class HelloController {

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    public String hello(String name) {
        return "Hello, " + name;
    }

    @RequestMapping("/")
    public ModelAndView index(ModelAndView mv) {
        mv.setViewName("index");
        mv.addObject("requestname", "This is index");
        return mv;
    }

    @RequestMapping("/rightaway")
    public ModelAndView returnRightAway(ModelAndView mv) {
        mv.setViewName("index");
        mv.addObject("requestname","This request is RightawayApi");
        return mv;
    }

    @RequestMapping("/sleep")
    public ModelAndView returnSleep(ModelAndView mv) throws InterruptedException {
        Thread.sleep(2*1000);
        mv.setViewName("index");
        mv.addObject("requestname","This request is SleepApi"+",it will sleep 2s !");
        return mv;
    }
}
```

