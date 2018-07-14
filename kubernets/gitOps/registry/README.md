### 概述

这是关于在树莓派`arm64`位中使用`alpine`系统编译以及安装docker镜像仓库[distribution](https://github.com/docker/distribution.git)的过程。

### 安装

关于树莓派64位系统的安装不在本次操作中，具体安装过程参考其他教程。在开始以下操作之前，默认已经安装好`docker`，主要工作是参考官方的`Dockerfile`文件，准备好`arm64`系统的`Dockerfile。

##### amd64的`Dockerfile`

```shell
FROM golang:1.8-alpine

ENV DISTRIBUTION_DIR /go/src/github.com/docker/distribution
ENV DOCKER_BUILDTAGS include_oss include_gcs

ARG GOOS=linux
ARG GOARCH=amd64

RUN set -ex \
    && apk add --no-cache make git

WORKDIR $DISTRIBUTION_DIR
COPY . $DISTRIBUTION_DIR
COPY cmd/registry/config-dev.yml /etc/docker/registry/config.yml

RUN make PREFIX=/go clean binaries

VOLUME ["/var/lib/registry"]
EXPOSE 5000
ENTRYPOINT ["registry"]
CMD ["serve", "/etc/docker/registry/config.yml"
```

##### `arm64`的`Dockerfile`

```shell
FROM arm64v8/alpine
RUN mkdir /go
ENV PATH /go:$PATH
COPY registry /go/registry
COPY config.yml /etc/docker/registry/config.yml
VOLUME ["/var/lib/registry"]
EXPOSE 5000
CMD ["registry", "serve", "/etc/docker/registry/config.yml"]
```

为了减轻`registry`的镜像大小，本次操作是先在`go`环境中编译出`registry`后在`arm64v8/alpine`环境中运行，编译后的镜像大小为`29.6`MB。

```shell
root@node1 registry]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
registry            latest              deeaf5fb0d25        22 minutes ago      29.6MB
arm64v8/alpine      latest              f6df4608fa98        3 months ago        3.96MB
```

##### `registry`编译

首先准备alpine的golang环境，通过`docker pull golang:alpine`，默认使用最新版本的`golang`，如想使用其他版本自行添加`tag`，之后通过`docker run -it --rm golang:alpine sh`进入编译系统，如下

```shell
[root@node1 registry]# docker run -it --rm golang:alpine sh
/go # ls
bin  src
/go/bin # cat /etc/apk/repositories
https://mirrors.aliyun.com/alpine/v3.7/main/
https://mirrors.aliyun.com/alpine/v3.7/community/
/go/bin # apk update
fetch https://mirrors.aliyun.com/alpine/v3.7/main/aarch64/APKINDEX.tar.gz
fetch https://mirrors.aliyun.com/alpine/v3.7/community/aarch64/APKINDEX.tar.gz
v3.7.0-127-g4b0114024e [https://mirrors.aliyun.com/alpine/v3.7/main/]
v3.7.0-117-g9584b2309e [https://mirrors.aliyun.com/alpine/v3.7/community/]
OK: 8861 distinct packages available
/go # apk add git gcc make musl-dev # 安装编译器等
/go/bin # go get -u -v github.com/docker/distribution # 下载源码
/go/src/github.com/docker/distribution # pwd # 进入源码路径
/go/src/github.com/docker/distribution
/go/src/github.com/docker/distribution # make PREFIX=/go clean binaries # 编译命令
```

> **修改apk软件源为阿里云软件源，加速软件安装速度，默认是通过官网进行下载安装**

在上述编译过程中会报一些底层库没有，如下所示

```shell
/opt/mygo/src/github.com/docker/distribution # make PREIFX=/opt/mygo clean binaries
+ clean
+ /opt/mygo/src/github.com/docker/distribution/bin/registry
# github.com/docker/distribution/cmd/registry
/usr/lib/go/pkg/tool/linux_arm64/link: running gcc failed: exit status 1
/usr/lib/gcc/aarch64-alpine-linux-musl/6.4.0/../../../../aarch64-alpine-linux-musl/bin/ld: cannot find Scrt1.o: No such file or directory
/usr/lib/gcc/aarch64-alpine-linux-musl/6.4.0/../../../../aarch64-alpine-linux-musl/bin/ld: cannot find crti.o: No such file or directory
/usr/lib/gcc/aarch64-alpine-linux-musl/6.4.0/../../../../aarch64-alpine-linux-musl/bin/ld: cannot find -lpthread
/usr/lib/gcc/aarch64-alpine-linux-musl/6.4.0/../../../../aarch64-alpine-linux-musl/bin/ld: cannot find -lssp_nonshared
/usr/lib/gcc/aarch64-alpine-linux-musl/6.4.0/../../../../aarch64-alpine-linux-musl/bin/ld: cannot find -lc
/usr/lib/gcc/aarch64-alpine-linux-musl/6.4.0/../../../../aarch64-alpine-linux-musl/bin/ld: cannot find crtn.o: No such file or directory
```

在该日志中提示找不到一些库，通过使用`find`命令查看的时候发现是存在

```shell
/usr/aarch64-alpine-linux-musl/bin # find / -name "crtn*"
/usr/lib/crtn.o
/usr/aarch64-alpine-linux-musl/bin # find / -name "crti.o"
/usr/lib/crti.o
/usr/aarch64-alpine-linux-musl/bin # find / -name "Scrt1.o"
/usr/lib/Scrt1.o
```

只是路径不对，需要在对应路径添加链接文件，让`make`能识别就行。

```shell
/usr/aarch64-alpine-linux-musl/bin # ln -s /usr/lib/Scrt1.o /usr/aarch64-alpine-linux-musl/bin/Scrt1.o
/usr/aarch64-alpine-linux-musl/bin # ln -s /usr/lib/crti.o /usr/aarch64-alpine-linux-musl/bin/crti.o
/usr/aarch64-alpine-linux-musl/bin # ln -s /usr/lib/crtn.o /usr/aarch64-alpine-linux-musl/bin/crtn.o
```

之后再次尝试编译，编译通过。

```shell
/go/src/github.com/docker/distribution # make PREFIX=/go clean binaries
+ clean
+ /go/bin/registry
+ /go/bin/digest
+ /go/bin/registry-api-descriptor-template
+ binaries
```

最终，我们有了编译好的二进制文件，通过`docker cp`将容器里面的二进制文件以及一些配置文件拷贝出来

```shell
[root@node1 registry]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
b61c4676f986        golang:alpine       "sh"                12 minutes ago      Up 12 minutes                           flamboyant_lewin
[root@node1 registry]# docker cp b61c4676f986:/go/bin .
[root@node1 registry]# docker build -t registry .
Sending build context to Docker daemon  36.21MB
Step 1/8 : FROM arm64v8/alpine
 ---> f6df4608fa98
Step 2/8 : RUN mkdir /go
 ---> Using cache
 ---> f2c89747c22e
Step 3/8 : ENV PATH /go:$PATH
 ---> Using cache
 ---> e9628c34ad1d
Step 4/8 : COPY registry /go/registry
 ---> 2555adf79abb
Step 5/8 : COPY config.yml /etc/docker/registry/config.yml
 ---> 81c5f14e422e
Step 6/8 : VOLUME ["/var/lib/registry"]
 ---> Running in a54913ff245c
Removing intermediate container a54913ff245c
 ---> 327f0c6fb46b
Step 7/8 : EXPOSE 5000
 ---> Running in 7db435d4d7ed
Removing intermediate container 7db435d4d7ed
 ---> 7592bb12d93f
Step 8/8 : CMD ["registry", "serve", "/etc/docker/registry/config.yml"]
 ---> Running in a0b7fc09303c
Removing intermediate container a0b7fc09303c
 ---> deeaf5fb0d25
Successfully built deeaf5fb0d25
Successfully tagged registry:latest
[root@node1 registry]# docker run -d --name registry -p 5000:5000 -v /data/registry:/var/lib/registry registry
da12cf86016071e39766d789a064cfa68d89d3324e17f2960315c2c01efd8eb8
[root@node1 registry]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
da12cf860160        registry            "registry serve /etc…"   4 seconds ago       Up 1 second         0.0.0.0:5000->5000/tcp   registry
b61c4676f986        golang:alpine       "sh"                     14 minutes ago      Up 14 minutes                                flamboyant_lewin
```

最后，我们通过`curl`发送`http`请求验证其功能正常

```shell
[root@node1 registry]# curl http://localhost:5000/v2/_catalog
{"repositories":[]}
```

### `registry`使用

