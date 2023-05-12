
##### 如何在java项目中使用maven

###### 为什么需要maven

考虑一个常见的场景：以项目A为例，开发过程中，需要依赖B-2.0.jar的包，如果没有maven，那么正常做法是把B-2.0.jar拷贝到项目A中，但是如果B-2.0.jar还依赖C.jar，我们还需要去找到C.jar的包，因此，在开发阶段需要花费在项目依赖方面的精力会很大。

因此，开发人员需要找到一种方式，可以管理java包的依赖关系，并可以方便的引入到项目中。

###### maven如何工作

查看`pom.xml`

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
```

可以直接在项目中添加上`dependency` ，这样来指定项目的依赖包。

思考：如果`spring-boot-starter-thymeleaf`包依赖别的包，怎么办？

`spring-boot-starter-thymeleaf`同时也是一个maven项目，也有自己的pom.xml

查看一下：

```xml
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
      <version>2.3.3.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.thymeleaf</groupId>
      <artifactId>thymeleaf-spring5</artifactId>
      <version>3.0.11.RELEASE</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.thymeleaf.extras</groupId>
      <artifactId>thymeleaf-extras-java8time</artifactId>
      <version>3.0.4.RELEASE</version>
      <scope>compile</scope>
    </dependency>
  </dependencies>
```

这样的话，使用maven的项目，只需要在自己的pom.xml中把所需的最直接的依赖包定义上，而不用关心这些被依赖的jar包自身是否还有别的依赖。剩下的都交给maven去搞定。

如何搞定？maven可以根据pom.xml中定义的依赖实现包的查找

去哪查找？maven仓库，存储jar包的地方。

当我们执行 Maven 构建命令时，Maven 开始按照以下顺序查找依赖的库：

![](images\maven-repo.png)

本地仓库：

- Maven 的本地仓库，在安装 Maven 后并不会创建，它是在第一次执行 maven 命令的时候才被创建。

- 运行 Maven 的时候，Maven 所需要的任何包都是直接从本地仓库获取的。如果本地仓库没有，它会首先尝试从远程仓库下载构件至本地仓库，然后再使用本地仓库的包。

- 默认情况下，不管Linux还是 Windows，每个用户在自己的用户目录下都有一个路径名为 .m2/respository/ 的仓库目录。

- Maven 本地仓库默认被创建在 %USER_HOME% 目录下。要修改默认位置，在 %M2_HOME%\conf 目录中的 Maven 的 settings.xml 文件中定义另一个路径。

  ```xml
  <settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
    <localRepository>D:\opt\maven-repo</localRepository>
  </settings>
  ```

中央仓库：

Maven 中央仓库是由 Maven 社区提供的仓库，中央仓库包含了绝大多数流行的开源Java构件，以及源码、作者信息、SCM、信息、许可证信息等。一般来说，简单的Java项目依赖的构件都可以在这里下载到。

中央仓库的关键概念：

- 这个仓库由 Maven 社区管理。
- 不需要配置，maven中集成了地址 http://repo1.maven.org/maven2 
- 需要通过网络才能访问。

私服仓库：

通常使用 sonatype Nexus来搭建私服仓库。搭建完成后，需要在 setting.xml中进行配置，比如：

```xml
<profile>
    <id>localRepository</id>
    <repositories>
        <repository>
            <id>myRepository</id>
            <name>myRepository</name>
            <url>http://127.0.0.1:8081/nexus/content/repositories/myRepository/</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </repository>
    </repositories>
</profile>
```



方便起见，我们直接使用国内ali提供的仓库，修改 maven 根目录下的 conf 文件夹中的 setting.xml 文件，在 mirrors 节点上，添加内容如下： 

```xml
<mirrors>
    <mirror>
      <id>alimaven</id>
      <name>aliyun maven</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>        
    </mirror>
</mirrors>
```

在执行构建的时候，maven会自动将所需的包下载到本地仓库中，所以第一次构建速度通常会慢一些，后面速度则很快。

那么maven是如何找到对应的jar包的？

我们可以访问 https://mvnrepository.com/ 查看在仓库中的jar包的样子。

```xml
<!-- https://mvnrepository.com/artifact/commons-collections/commons-collections -->
<dependency>
    <groupId>commons-collections</groupId>
    <artifactId>commons-collections</artifactId>
    <version>3.2.2</version>
</dependency>
```

刚才看到`spring-boot-starter-thymeleaf`的依赖同样有上述属性，因此maven就可以根据这三项属性，到对应的仓库中去查找到所需要的依赖包，并下载到本地。

其中`groupId、artifactId、version`共同保证了包在仓库中的唯一性，这也就是为什么maven项目的pom.xml中都先配置这几项的原因，因为项目最终发布到远程仓库中，供别人调用。

思考：我们项目的`dependency`中为什么没有写`version` ?

是因为sprintboot项目的`上面有人`，来看一下项目`parent`的写法：

```xml
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
```

parent模块中定义过的`dependencies`，在子项目中引用的话，不需要指定版本，这样可以保证所有的子项目都使用相同版本的依赖包。

###### 生命周期及mvn命令实践

 Maven有三套相互独立的生命周期，分别是clean、default和site。每个生命周期包含一些阶段（phase），阶段是有顺序的，后面的阶段依赖于前面的阶段。 

- clean生命周期，清理项目
  - 清理：mvn clean　　　 --删除target目录，也就是将class文件等删除
- default生命周期，项目的构建等核心阶段
  - 编译：mvn compile　　--src/main/java目录java源码编译生成class （target目录下）
  - 测试：mvn test　　　　--src/test/java 执行目录下的测试用例
  - 打包：mvn package　　--生成压缩文件：java项目#jar包；web项目#war包，也是放在target目录下
  - 安装：mvn install　　　--将压缩文件(jar或者war)上传到本地仓库
  - 部署|发布：mvn deploy　　--将压缩文件上传私服
- site生命周期，建立和发布项目站点
  - 站点 : mvn site      --生成项目站点文档

各个生命周期相互独立，一个生命周期的阶段前后依赖。 生命周期阶段需要绑定到某个插件的目标才能完成真正的工作，比如test阶段正是与maven-surefire-plugin的test目标相绑定了 。

举例如下：

- mvn clean

  调用clean生命周期的clean阶段

- mvn test

  调用default生命周期的test阶段，实际执行test以及之前所有阶段

- mvn clean install

  调用clean生命周期的clean阶段和default的install阶段，实际执行clean，install以及之前所有阶段

在linux环境中演示：

创建gitlab组，`luffy-spring-cloud`,在该组下创建项目`springboot-demo`

- 提交代码到git仓库

  ```bash
  $ git init
  $ git remote add origin http://gitlab.luffy.com/luffy-spring-cloud/springboot-demo.git
  $ git add .
  $ git commit -m "Initial commit"
  $ git push -u origin master
  ```

- 使用tools容器来运行

  ```bash
  $ docker run --rm -ti 172.21.51.143:5000/devops/tools:v3 bash
  bash-5.0# mvn -v
  bash: mvn: command not found
  # 由于idea工具自带了maven，所以可以直接在ide中执行mvn命令。在tools容器中，需要安装mvn命令
  ```

  为tools镜像集成mvn：

  将本地的`apache-maven-3.6.3`放到`tools`项目中，修改`settings.xml`配置

  ```xml
  ...
  <localRepository>/opt/maven-repo</localRepository>
  ...
  ```

  修改mirrors：

  ```bash
  <mirrors>
      <mirror>
        <id>alimaven</id>
        <name>aliyun maven</name>
        <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
        <mirrorOf>central</mirrorOf>        
      </mirror>
  </mirrors>
  ```

  

  然后修改Dockerfile，添加如下部分:

  ```dockerfile
  #-----------------安装 maven--------------------#
  COPY apache-maven-3.6.3 /usr/lib/apache-maven-3.6.3
  RUN ln -s /usr/lib/apache-maven-3.6.3/bin/mvn /usr/local/bin/mvn && chmod +x /usr/local/bin/mvn
  ENV MAVEN_HOME=/usr/lib/apache-maven-3.6.3
  #------------------------------------------------#
  ```



去master节点拉取最新代码，构建最新的tools镜像：

```bash
  # k8s-master节点
  $ git pull
  $ docker build . -t 172.21.51.143:5000/devops/tools:v4 -f Dockerfile
  $ docker push 172.21.51.143:5000/devops/tools:v4
```

  再次尝试mvn命令：

  ```bash
$ docker run -v /var/run/docker.sock:/var/run/docker.sock --rm -ti 172.21.51.143:5000/devops/tools:v4 bash
  bash-5.0# mvn -v
  bash-5.0# git clone http://gitlab.luffy.com/luffy-spring-cloud/springboot-demo.git
  bash-5.0# cd springboot-demo
  bash-5.0# mvn clean
  # 观察/opt/maven目录
  bash-5.0# mvn package
  # 多阶段组合
  bash-5.0# mvn clean package
  ```

  

想系统学习maven，可以参考： https://www.runoob.com/maven/maven-pom.html 

