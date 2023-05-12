###### 多阶构建

https://gitee.com/agagin/href-counter.git

原始构建：

```dockerfile
FROM golang:1.13

WORKDIR /go/src/github.com/alexellis/href-counter/

COPY vendor vendor
COPY app.go .
ENV GOPROXY https://goproxy.cn
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

CMD ["./app"]
```

```bash
$ docker build . -t href-counter:v1 -f Dockerfile
```



多阶构建：

```dockerfile
FROM golang:1.13 AS builder

WORKDIR /go/src/github.com/alexellis/href-counter/

COPY vendor vendor
COPY app.go	.
ENV GOPROXY https://goproxy.cn

RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

FROM alpine:3.10
RUN apk --no-cache add ca-certificates

WORKDIR /root/

COPY --from=builder  /go/src/github.com/alexellis/href-counter/app    .

CMD ["./app"]
```

```bash
$ docker build . -t href-counter:v2 -f Dockerfile.multi
```


https://gitee.com/agagin/springboot-app.git

原始构建：

```dockerfile
FROM srinivasansekar/javamvn

WORKDIR /opt/springboot-app
COPY  . .
RUN mvn clean package -DskipTests=true

CMD [ "sh", "-c", "java -jar /opt/springboot-app/target/sample.jar" ]
```

```bash
$ docker build . -t sample:v1 -f Dockerfile
```



多阶构建：

```dockerfile
FROM maven as builder

WORKDIR /opt/springboot-app
COPY  . .
RUN mvn clean package -DskipTests=true

FROM openjdk:8-jdk-alpine
COPY --from=builder /opt/springboot-app/target/sample.jar sample.jar
CMD [ "sh", "-c", "java -jar /sample.jar" ]
```

```bash
$ docker build . -t sample:v2 -f Dockerfile.multi
```



原则：

- 不必要的内容不要放在镜像中
- 减少不必要的层文件
- 减少网络传输操作
- 可以适当的包含一些调试命令
