###### Dockerfile使用

```bash
$ docker build . -t ImageName:ImageTag -f Dockerfile
```

Dockerfile是一堆指令，在docker build的时候，按照该指令进行操作，最终生成我们期望的镜像

- FROM 指定基础镜像，必须为第一个命令

  ```
  格式：
  	FROM <image>
  	FROM <image>:<tag>
  示例：
  	FROM mysql:5.7
  注意：
  	tag是可选的，如果不使用tag时，会使用latest版本的基础镜像
  ```

- MAINTAINER 镜像维护者的信息

  ```
  格式：
  	MAINTAINER <name>
  示例：
  	MAINTAINER Yongxin Li
      MAINTAINER inspur_lyx@hotmail.com
      MAINTAINER Yongxin Li <inspur_lyx@hotmail.com>
  ```

- COPY|ADD 添加本地文件到镜像中

  ```
  格式：
  	COPY <src>... <dest>
  示例：
      ADD hom* /mydir/          # 添加所有以"hom"开头的文件
      ADD test relativeDir/     # 添加 "test" 到 `WORKDIR`/relativeDir/
      ADD test /absoluteDir/    # 添加 "test" 到 /absoluteDir/
  ```

- WORKDIR 工作目录

  ```
  格式：
  	WORKDIR /path/to/workdir
  示例：
      WORKDIR /a  (这时工作目录为/a)
  注意：
  	通过WORKDIR设置工作目录后，Dockerfile中其后的命令RUN、CMD、ENTRYPOINT、ADD、COPY等命令都会在该目录下执行
  ```

- RUN 构建镜像过程中执行命令

  ```
  格式：
  	RUN <command>
  示例：
      RUN yum install nginx
      RUN pip install django
      RUN mkdir test && rm -rf /var/lib/unusedfiles
  注意：
  	RUN指令创建的中间镜像会被缓存，并会在下次构建中使用。如果不想使用这些缓存镜像，可以在构建时指定--no-cache参数，如：docker build --no-cache
  ```

- CMD 构建容器后调用，也就是在容器启动时才进行调用

  ```
  格式：
      CMD ["executable","param1","param2"] (执行可执行文件，优先)
      CMD ["param1","param2"] (设置了ENTRYPOINT，则直接调用ENTRYPOINT添加参数)
      CMD command param1 param2 (执行shell内部命令)
  示例：
      CMD ["/usr/bin/wc","--help"]
      CMD ping www.baidu.com
  注意：
  	CMD不同于RUN，CMD用于指定在容器启动时所要执行的命令，而RUN用于指定镜像构建时所要执行的命令。
  ```

- ENTRYPOINT 设置容器初始化命令，使其可执行化

  ```
  格式：
      ENTRYPOINT ["executable", "param1", "param2"] (可执行文件, 优先)
      ENTRYPOINT command param1 param2 (shell内部命令)
  示例：
      ENTRYPOINT ["/usr/bin/wc","--help"]
  注意：
  	ENTRYPOINT与CMD非常类似，不同的是通过docker run执行的命令不会覆盖ENTRYPOINT，而docker run命令中指定的任何参数，都会被当做参数再次传递给ENTRYPOINT。Dockerfile中只允许有一个ENTRYPOINT命令，多指定时会覆盖前面的设置，而只执行最后的ENTRYPOINT指令
  ```

- ENV

  ```
  格式：
      ENV <key> <value>
      ENV <key>=<value>
  示例：
      ENV myName John
      ENV myCat=fluffy
  ```

- EXPOSE

  ```
  格式：
      EXPOSE <port> [<port>...]
  示例：
      EXPOSE 80 443
      EXPOSE 8080
      EXPOSE 11211/tcp 11211/udp
  注意：
      EXPOSE并不会让容器的端口访问到主机。要使其可访问，需要在docker run运行容器时通过-p来发布这些端口，或通过-P参数来发布EXPOSE导出的所有端口
  
  ```

  ![](images/Dockerfile解释.png)



- 基础环境镜像

  ```dockerfile
  FROM java:8-alpine
  
  RUN apk add --update ca-certificates && rm -rf /var/cache/apk/* && \
    find /usr/share/ca-certificates/mozilla/ -name "*.crt" -exec keytool -import -trustcacerts \
    -keystore /usr/lib/jvm/java-1.8-openjdk/jre/lib/security/cacerts -storepass changeit -noprompt \
    -file {} -alias {} \; && \
    keytool -list -keystore /usr/lib/jvm/java-1.8-openjdk/jre/lib/security/cacerts --storepass changeit
  
  ENV MAVEN_VERSION 3.5.4
  ENV MAVEN_HOME /usr/lib/mvn
  ENV PATH $MAVEN_HOME/bin:$PATH
  
  RUN wget http://archive.apache.org/dist/maven/maven-3/$MAVEN_VERSION/binaries/apache-maven-$MAVEN_VERSION-bin.tar.gz && \
    tar -zxvf apache-maven-$MAVEN_VERSION-bin.tar.gz && \
    rm apache-maven-$MAVEN_VERSION-bin.tar.gz && \
    mv apache-maven-$MAVEN_VERSION /usr/lib/mvn
  
  RUN mkdir -p /usr/src/app
  WORKDIR /usr/src/app
  
  ```

- 前端镜像

  ```dockerfile
  FROM nginx:1.19.0-alpine
  
  LABEL maintainer="mritd <mritd@linux.com>"
  
  ARG TZ='Asia/Shanghai'
  ENV TZ ${TZ}
  
  RUN apk upgrade --update \
      && apk add bash tzdata curl wget ca-certificates \
      && ln -sf /usr/share/zoneinfo/${TZ} /etc/localtime \
      && echo ${TZ} > /etc/timezone \
      && rm -rf /usr/share/nginx/html /var/cache/apk/*
  
  COPY dist /usr/share/nginx/html
  
  EXPOSE 80 443
  
  CMD ["nginx", "-g", "daemon off;"]
  ```

- java镜像

  ```dockerfile
  FROM java:8u111
  
  ENV JAVA_OPTS "\
  -Xmx4096m \
  -XX:MetaspaceSize=256m \
  -XX:MaxMetaspaceSize=256m"
  ENV JAVA_HOME /usr/java/jdk
  ENV PATH ${PATH}:${JAVA_HOME}/bin
  
  COPY target/myapp.jar myapp.jar
  
  RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
  RUN echo 'Asia/Shanghai' >/etc/timezone
  
  EXPOSE 9000
  CMD java ${JAVA_OPTS} -jar myapp.jar
  ```

  

- golang镜像

  多阶段构建


###### 通过1号进程理解容器的本质

```bash
$ docker exec -ti my-nginx-alpine /bin/sh
#/ ps aux

```

容器启动的时候可以通过命令去覆盖默认的CMD

```bash
$ docker run -d --name xxx nginx:alpine <自定义命令>
# <自定义命令>会覆盖镜像中指定的CMD指令，作为容器的1号进程启动。

$ docker run -d --name test-3 nginx:alpine echo 123

$ docker run -d --name test-4 nginx:alpine ping www.luffycity.com

```



本质上讲容器是利用namespace和cgroup等技术在宿主机中创建的独立的虚拟空间，这个空间内的网络、进程、挂载等资源都是隔离的。

```bash
$ docker exec -ti my-nginx /bin/sh
#/ ip addr
#/ ls -l /
#/ apt install xxx
#/ #安装的软件对宿主机和其他容器没有任何影响，和虚拟机不同的是，容器间共享一个内核，所以容器内没法升级内核
```


