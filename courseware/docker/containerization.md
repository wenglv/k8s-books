
#### Django应用容器化实践

###### django项目介绍

- 项目地址：https://gitee.com/agagin/python-demo.git
- python3 + django + uwsgi + nginx + mysql

- 内部服务端口8002

###### 容器化Django项目

*dockerfiles/myblog/Dockerfile*

```dockerfile
# This my first django Dockerfile
# Version 1.0

# Base images 基础镜像
FROM centos:centos7.5.1804

#MAINTAINER 维护者信息
LABEL maintainer="inspur_lyx@hotmail.com"

#ENV 设置环境变量
ENV LANG en_US.UTF-8
ENV LC_ALL en_US.UTF-8

#RUN 执行以下命令
RUN curl -so /etc/yum.repos.d/Centos-7.repo http://mirrors.aliyun.com/repo/Centos-7.repo && rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
RUN yum install -y  python36 python3-devel gcc pcre-devel zlib-devel make net-tools nginx

#工作目录
WORKDIR /opt/myblog

#拷贝文件至工作目录
COPY . .

# 拷贝nginx配置文件
COPY myblog.conf /etc/nginx

#安装依赖的插件
RUN pip3 install -i http://mirrors.aliyun.com/pypi/simple/ --trusted-host mirrors.aliyun.com -r requirements.txt

RUN chmod +x run.sh && rm -rf ~/.cache/pip

#EXPOSE 映射端口
EXPOSE 8002

#容器启动时执行命令
CMD ["./run.sh"]
```

执行构建：

```bash
$ docker build . -t myblog:v1 -f Dockerfile

```

###### 运行mysql

```bash
$ docker run -d -p 3306:3306 --name mysql  -v /opt/mysql:/var/lib/mysql -e MYSQL_DATABASE=myblog -e MYSQL_ROOT_PASSWORD=123456 mysql:5.7 --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci

## 参数传递
## 查看数据库
$ docker exec -ti mysql bash
#/ mysql -uroot -p123456
#/ show databases;

## navicator连接

```

###### 启动Django应用

```bash
## 启动容器
$ docker run -d -p 8002:8002 --name myblog -e MYSQL_HOST=172.21.51.143 -e MYSQL_USER=root -e MYSQL_PASSWD=123456  myblog:v1 

## migrate
$ docker exec -ti myblog bash
#/ python3 manage.py makemigrations
#/ python3 manage.py migrate
#/ python3 manage.py createsuperuser

## 创建超级用户
$ docker exec -ti myblog python3 manage.py createsuperuser

## 收集静态文件
## $ docker exec -ti myblog python3 manage.py collectstatic

```

访问172.21.51.143:8002/admin

