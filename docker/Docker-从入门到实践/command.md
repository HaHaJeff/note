# Dockerfile指令详解

## COPY复制文件
**格式：**
- COPY <源路径>...<目标路径>
- COPY ["<源路径1>"，... "[目标路径]"]

```COPY```指令将从构建上下文目录中``` <源路径> ```的文件/目录复制到新的一层的镜像内的``` <目标路径> ```位置。比如：
```
COPY package.json /usr/src/app/
```
- ``` <源路径> ```可以是多个，甚至可以是通配符，其通配符规则要满足Go的```filepath.Match```规则。
- ``` <目标路径>``` 可以是容器内的绝对路径，也可以是相对于工作目录的相对路径(工作目录可以用 ```WORKDIR ```指令来指定)。目标路径不需要事先创建，如果目录不存在会复制文件前线性创建缺失目录。

**NOTE：使用```COPY```指令，源文件的各种元数据都会保留**

## ADD更高级的复制文件
``` ADD ```指令和``` COPY ```的格式和性质基本一直。但是在```COPY```基础上增加了一些功能。
比如``` <源路径> ``` 可以是一个``` URL ```，这种情况下，Docker引擎会试图去下载这个链接并放到 ``` <目标路径> ```中。**下载后的文件权限自动设置为``` 600 ``` ，如果这并不是想要的权限那么该需要增加一层额外的``` RUN ```进行权限调整，另外如果下载的是个压缩包，需要解压缩，也一样还需要增加一层额外的``` RUN ```指令进行解压缩。**所以不如直接使用``` RUN ```命令，然后使用``` wget ```或``` curl ```工具下载，处理权限、解压缩、然后清理无用文件更合理。

如果 ``` <源路径> ```为一个 ``` tar ```压缩文件的话，压缩格式为``` gzip ```，``` bzip2 ```以及 ``` xz ```的情况下， ``` ADD ```指令将会自动解压缩这个压缩文件到 ``` <目标路径> ```去。

在Docker官方的[Dockerfile最佳实践文档](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#use-a-dockerignore-file)中要求，尽可能的使用``` COPY ```，因为 ``` COPY ```的语义很明确，就是复制文件而已，而``` ADD ```则包含了更复杂的功能，其行为也不一定很清晰。**NOTE：ADD指令会令镜像构建缓存失效，从而可能会令镜像构建变得缓慢。**所以在```COPY```和```ADD```指令中进行选择的时候，所有的文件复制均用```COPY```指令，仅在需要自动解压缩的场景使用```ADD```。

## CMD容器启动命令
- ```shell```格式：CMD <命令>
- ``` exec ```格式：CMD ["可执行文件"，"参数1"，"参数2"...]
- 参数列表格式：CMD ["参数1"，"参数2"...]。在指定了``` ENTRYPOINT ```指令后，用``` CMD ```指定具体的参数。

**Docker不是虚拟机，容器就是进程。既然是进程，那么在启动容器的时候，需要指定所运行的程序以及参数。``` CMD ```指令用于指定默认的容器主进程的启动命令的。**

## ENTRYPOINT入口点
``` ENTRYPOINT ```的格式也分为``` exec ```和 ``` shell ```格式。

``` ENTRYPOINT ```的目的和 ``` CMD ```一样，都是在指定容器启动程序以及参数。 ``` ENTRYPOINT ```在运行时也可以替代，不过比 ``` CMD ```要略显繁琐，需要通过 ``` docker run ```的参数 ``` --entrypoint ```来指定。

当指定了``` ENTRYPOINT ```后，``` CMD ```的含义就发生了改变，不再是直接的而运行其命令，而是将``` CMD ```的内容作为参数传给``` ENTRYPOINT ```指令。

## ENV环境变量
- ENV < key > <  value >
- ENV < key1 >=< value1 > < key2 >=< value2 >

可以直接支持环境变量展开的指令：
```
ADD、COPY、ENV、EXPOSE、LABEL、USER、WORKDIR、VOLUME、STOPSIGNAL、ONBUILD 
```


## ARG构建参数
- ARG <参数名>[=<默认值>]

构建参数和``` ENV ```效果一样，都是设置环境变量。所不同的是，``` ARG ```所设置的构建环境的环境变量，在将来容器运行时不会存在这些环境变量的。但是不要因此使用``` ARG ```保存密码之类的信息，因为``` docker history``` 还是可以看到所有值得。

## VOLUME定义匿名卷
- VOLUME ["<路径1>"，"<路径2>"...]
- VOLUME <路径>

**NOTE：容器运行时应该尽量保持容器存储层不发生写操作，对于数据路类需要保存动态数据的应用，应该将其数据库文件保存于卷(VOLUME)中**。

## EXPOSE声明端口
``` EXPOSE <端口1> [<端口2>...]```。
```EXPOSE```指令时声明运行时容器提供服务端口，仅仅只是一个声明，在运行时并不会因为这个声明就开启这个端口。
- 帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射；
- 在运行时使用随机端口映射时，也就是```docker run -P```时，会自动随机映射```EXPOSE```指定的端口。


## WORKDIR指定工作目录
格式为```WORKDIR <工作目录路径>```
使用```WORKDIR```指令可以来指定工作目录，以后各层的当前目录就被改为指定的目录，如该目录不存在，```WORKDIR```会帮你建立该目录。


## USER指定当前用户
格式为```USER <用户名>```
```USER```和```WORKDIR```相似，都是改变环境状态并影响以后的层。```WORKDIR```是改变工作目录，```USER```则是改变之后层的执行```RUN```，```CMD```以及```ENTRYPOINT```这类命令的身份。


# 多阶段构建

## 全部放入一个Dockerfile
一种方式是将所有的构建过程包含在一个Dockerfile中，包括项目以及其依赖库的编译，测试，打包等流程，这里可能会带来一些问题：
- ```Dockerfile```特别长，可维护性降低；
- 镜像层次多，镜像体积较大，部署时间变长；
- 源代码存在泄露的风险。
```
FROM golang:1.9-alpine
RUN apk --no-cache add git ca-certificates
WORKDIR /go/src/github.com/go/helloworld/
COPY app.go .
RUN go get -d -v github.com/go-sql-driver/mysql \
&& CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app . \
&& cp /go/src/github.com/go/helloworld/app /root
WORKDIR /root/
CMD ["./app"]
```

## 分散到多个Dockerfile
另外一种方式，就是我们事先在一个```Dockerfile```将项目及其依赖编译打包好后，再将其拷贝到运行环境中，这种方式需要编写两个```Dockerfile```和一些编译脚本才能将其两个阶段自动整合起来。

**编写```Dockerfile.build```文件：**
```
FROM golang:1.9-alpine
RUN apk --no-cache add git
WORKDIR /go/src/github.com/go/helloworld
COPY app.go .
RUN go get -d -v github.com/go-sql-driver/mysql \
&& CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .
```

**编写```Dockerfile.copy```文件：**
```
FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY app .
CMD ["./app"]
```

**编写```build.sh```文件：**
```
#!/bin/sh
echo Building go/helloworld:build
docker build -t go/helloworld:build . -f Dockerfile.build
docker create --name extract go/helloworld:build
docker cp extract:/go/src/github.com/go/helloworld/app ./app
docker rm -f extract
echo Building go/helloworld:2
docker build --no-cache -t go/helloworld:2 . -f Dockerfile.copy
rm ./app
```

## 使用多阶段构建
```
FROM golang:1.9-alpine
RUN apk --no-cache add git
WORKDIR /go/src/github.com/go/helloworld/
RUN go get -d -v github.com/go-sql-driver/mysql
COPY app.go .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .
FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=0 /go/src/github.com/go/helloworld/app .
CMD ["./app"]
```
**对比三种不同镜像的构建方式：**
```
$ docker image ls
REPOSITORY TAG IMAGE ID CREATED SIZE
go/helloworld 3 d6911ed9c846 7 seconds ago 6.47MB
go/helloworld 2 f7cf3465432c 22 seconds ago 6.47MB
go/helloworld 1 f55d3e16affc 2 minutes ago 295M
```