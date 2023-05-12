#### 核心要素及常用操作详解

![](images/docker架构.png)

三大核心要素：镜像(Image)、容器(Container)、仓库(Registry)

###### 镜像（Image）

打包了业务代码及运行环境的包，是静态的文件，不能直接对外提供服务。

###### 容器（Container）

镜像的运行时，可以对外提供服务。

###### 仓库（Registry）

存放镜像的地方

- 公有仓库，Docker Hub，阿里，网易...
- 私有仓库，企业内部搭建
  - Docker Registry，Docker官方提供的镜像仓库存储服务
  - Harbor, 是Docker Registry的更高级封装，它除了提供友好的Web UI界面，角色和用户权限管理，用户操作审计等功能 
- 镜像访问地址形式 registry.devops.com/demo/hello:latest,若没有前面的url地址，则默认寻找Docker Hub中的镜像，若没有tag标签，则使用latest作为标签。 比如，docker pull nginx，会被解析成docker.io/library/nginx:latest
- 公有的仓库中，一般存在这么几类镜像
  - 操作系统基础镜像（centos，ubuntu，suse，alpine）
  - 中间件（nginx，redis，mysql，tomcat）
  - 语言编译环境（python，java，golang）
  - 业务镜像（django-demo...）

容器和仓库不会直接交互，都是以镜像为载体来操作。

1. 查看镜像列表

   ```bash
   $ docker images
   ```

2. 如何获取镜像

   - 从远程仓库拉取

     ```bash
     $ docker pull nginx:alpine
     $ docker images
     ```

   - 使用tag命令

     ```bash
     $ docker tag nginx:alpine 172.21.51.143:5000/nginx:alpine
     $ docker images
     ```

   - 本地构建

     ```bash
     $ docker build . -t my-nginx:ubuntu -f Dockerfile
     ```

3. 如何通过镜像启动容器

   ```bash
   $ docker run --name my-nginx-alpine -d nginx:alpine
   ```

4. 如何知道容器内部运行了什么程序？

   ```bash
   # 进入容器内部,分配一个tty终端
   $ docker exec -ti my-nginx-alpine /bin/sh
   # ps aux
   ```

5. docker怎么知道容器启动后该执行什么命令？

   通过docker build来模拟构建一个nginx的镜像，

   - 创建Dockerfile

     ```dockerfile
     # 告诉docker使用哪个基础镜像作为模板，后续命令都以这个镜像为基础 
     FROM ubuntu
     
     # RUN命令会在上面指定的镜像里执行命令 
     RUN apt-get update && apt install -y nginx
     
     #告诉docker，启动容器时执行如下命令
     CMD ["/usr/sbin/nginx", "-g","daemon off;"]
     ```

   - 构建本地镜像

     ```bash
     $ docker build . -t my-nginx:ubuntu -f Dockerfile
     ```

- 使用新镜像启动容器

  ```bash
     $ docker run --name my-nginx-ubuntu -d my-nginx:ubuntu
     
  ```

- 进入容器查看进程

  ```bash
     $ docker exec -ti my-nginx-ubuntu /bin/sh
     # ps aux
     
  ```

6. 如何访问容器内服务

   ```bash
   # 进入容器内部
   $ docker exec -ti my-nginx-alpine /bin/sh
   # ps aux|grep nginx
   # curl localhost:80
   
   ```

7. 宿主机中如何访问容器服务

   ```bash
   # 删掉旧服务,重新启动
   $ docker rm -f my-nginx-alpine
   $ docker run --name my-nginx-alpine -d -p 8080:80 nginx:alpine
   $ curl 172.21.51.143:8080
   ```

8. docker client如何与daemon通信

   ```bash
   # /var/run/docker.sock
   $ docker run --name portainer -d -p 9001:9000 -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer
   ```





###### 操作演示

![](images/常用命令.jpg)

1. 查看所有镜像：

```bash
$ docker images
```

2. 拉取镜像:

```bash
$ docker pull nginx:alpine
```

3. 如何唯一确定镜像:

- image_id
- repository:tag

```bash
$ docker images
REPOSITORY    TAG                 IMAGE ID            CREATED             SIZE
nginx         alpine              377c0837328f        2 weeks ago         19.7MB
```

4. 导出镜像到文件中

   ```bash
   $ docker save -o nginx-alpine.tar nginx:alpine
   
   ```

5. 从文件中加载镜像

   ```bash
   $ docker load -i nginx-alpine.tar
   
   ```

6. 部署镜像仓库

   https://docs.docker.com/registry/ 

   ```bash
   ## 使用docker镜像启动镜像仓库服务
   $ docker run -d -p 5000:5000 --restart always --name registry registry:2
   
   ## 默认仓库不带认证，若需要认证，参考https://docs.docker.com/registry/deploying/#restricting-access
   
   ```

7. 推送本地镜像到镜像仓库中

   ```bash
   $ docker tag nginx:alpine localhost:5000/nginx:alpine
   $ docker push localhost:5000/nginx:alpine
   
   ## 查看仓库内元数据
   $ curl -X GET http://172.21.51.143:5000/v2/_catalog
   $ curl -X GET http://172.21.51.143:5000/v2/nginx/tags/list
   
   ## 镜像仓库给外部访问，不能通过localhost，尝试使用内网地址172.21.51.143:5000/nginx:alpine
   $ docker tag nginx:alpine 172.21.51.143:5000/nginx:alpine
   $ docker push 172.21.51.143:5000/nginx:alpine
   The push refers to repository [172.21.51.143:5000/nginx]
   Get https://172.21.51.143:5000/v2/: http: server gave HTTP response to HTTPS client
   ## docker默认不允许向http的仓库地址推送，如何做成https的，参考：https://docs.docker.com/registry/deploying/#run-an-externally-accessible-registry
   ## 我们没有可信证书机构颁发的证书和域名，自签名证书需要在每个节点中拷贝证书文件，比较麻烦，因此我们通过配置daemon的方式，来跳过证书的验证：
   $ cat /etc/docker/daemon.json
   {
     "registry-mirrors": [
       "https://8xpk5wnt.mirror.aliyuncs.com"
     ],
     "insecure-registries": [
        "172.21.51.143:5000"
     ]
   }
   $ systemctl restart docker
   $ docker push 172.21.51.143:5000/nginx:alpine
   $ docker images	# IMAGE ID相同，等于起别名或者加快捷方式
   REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
   172.21.51.143:5000/nginx   alpine              377c0837328f        4 weeks ago         
   nginx                    alpine              377c0837328f        4 weeks ago         
   localhost:5000/nginx     alpine              377c0837328f        4 weeks ago         
   registry                 2                   708bc6af7e5e        2 months ago       
   
   ```

8. 删除镜像

   ```bash
   docker rmi nginx:alpine
   ```

9. 查看容器列表

   ```bash
   ## 查看运行状态的容器列表
   $ docker ps
   
   ## 查看全部状态的容器列表
   $ docker ps -a
   
   ```

10. 启动容器

    ```bash
    ## 后台启动
    $ docker run --name nginx -d nginx:alpine
    
    ## 映射端口,把容器的端口映射到宿主机中,-p <host_port>:<container_port>
    $ docker run --name nginx -d -p 8080:80 nginx:alpine
    
    ## 资源限制,最大可用内存500M
    $ docker run --memory=500m nginx:alpine
    
    ```

    

11. 容器数据持久化

     ```bash
     ## 挂载主机目录
     $ docker run --name nginx -d  -v /opt:/opt  nginx:alpine
     $ docker run --name mysql -e MYSQL_ROOT_PASSWORD=123456  -d -v /opt/mysql/:/var/lib/mysql mysql:5.7
     ```

12. 进入容器或者执行容器内的命令

     ```bash
     $ docker exec -ti <container_id_or_name> /bin/sh
     $ docker exec <container_id_or_name> hostname
    
     ```

13. 主机与容器之间拷贝数据

     ```bash
     ## 主机拷贝到容器
     $ echo '123'>/tmp/test.txt
     $ docker cp /tmp/test.txt nginx:/tmp
     $ docker exec -ti nginx cat /tmp/test.txt
     123
     
     ## 容器拷贝到主机
     $ docker cp nginx:/tmp/test.txt ./
    
     ```


14. 查看容器日志

     ```bash
     ## 查看全部日志
     $ docker logs nginx
     
     ## 实时查看最新日志
     $ docker logs -f nginx
     
     ## 从最新的100条开始查看
     $ docker logs --tail=100 -f nginx
    
     ```

15. 停止或者删除容器

     ```bash
     ## 停止运行中的容器
     $ docker stop nginx
     
     ## 启动退出容器
     $ docker start nginx
     
     ## 删除非运行中状态的容器
     $ docker rm nginx
     
     ## 删除运行中的容器
     $ docker rm -f nginx
    
     ```

16. 查看容器或者镜像的明细

     ```bash
     ## 查看容器详细信息，包括容器IP地址等
     $ docker inspect nginx
     
     ## 查看镜像的明细信息
     $ docker inspect nginx:alpine
    
     ```

