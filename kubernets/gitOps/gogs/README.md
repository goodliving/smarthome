### 概述

这是关于在树莓派64位系统中安装`git`服务器gogs，包括源码编译以及容器化。

### 准备工作

* 编译环境
* `Dockerfile`文件

##### 编译环境

首先准备`golang`的alpine系统环境，通过改造官方的golang镜像来搭建编译环境，如下所示

```shell
[root@drone-ci golang]# cat Dockerfile
FROM golang:alpine
RUN echo https://mirrors.aliyun.com/alpine/v3.7/main/ > /etc/apk/repositories
RUN echo https://mirrors.aliyun.com/alpine/v3.7/community/ >> /etc/apk/repositories
RUN apk update && \
        apk add --no-cache build-base git gcc make
RUN rm -rf /var/cache/apk/*
[root@drone-ci golang]# docker build -t antilope:alpine .
```

其中，在编译镜像中添加阿里软件源地址，之后安装编译器`gcc`，这样我们就有了`alpine`系统的golang，默认是最新的版本，如有版本要求，自行修改镜像`tag`。

然后，我们通过`docker run -it`进入编译环境，通过`go get`来下载`gogs`源码，推荐通过`git`从[码云仓库](https://gitee.com/Unknown/gogs)中下载到本地，需要注意的是保持目录格式一致，在此不做过多说明。如下所示

```shell
/go/src/github.com/gogits/gogs # l
total 92
-rw-r--r--    1 root     root           908 Mar 25 04:45 Dockerfile
-rw-r--r--    1 root     root           916 Mar 25 04:45 Dockerfile.aarch64
-rw-r--r--    1 root     root           914 Mar 25 04:45 Dockerfile.rpi
-rw-r--r--    1 root     root          1460 Mar 25 04:45 Dockerfile.rpihub
-rw-r--r--    1 root     root          1054 Mar 25 04:45 LICENSE
-rw-r--r--    1 root     root          1957 Mar 25 04:45 Makefile
-rw-r--r--    1 root     root          8032 Mar 25 04:45 README.md
-rw-r--r--    1 root     root          5329 Mar 25 04:45 README_ZH.md
-rw-r--r--    1 root     root           295 Mar 25 04:45 appveyor.yml
drwxr-xr-x    2 root     root          4096 Mar 25 04:45 cmd
drwxr-xr-x    7 root     root          4096 Mar 25 04:45 conf
drwxr-xr-x    4 root     root          4096 Mar 25 04:45 docker
-rw-r--r--    1 root     root           760 Mar 25 04:45 gogs.go
drwxr-xr-x    4 root     root          4096 Mar 25 04:45 models
drwxr-xr-x    3 root     root          4096 Mar 25 04:45 packager
drwxr-xr-x   18 root     root          4096 Mar 25 04:45 pkg
drwxr-xr-x    8 root     root          4096 Mar 25 04:45 public
drwxr-xr-x    8 root     root          4096 Mar 25 04:45 routes
drwxr-xr-x    7 root     root          4096 Mar 25 04:45 scripts
drwxr-xr-x   11 root     root          4096 Mar 25 04:45 templates
drwxr-xr-x    5 root     root          4096 Mar 25 04:45 vendor
/go/src/github.com/gogits/gogs # go build -o gogs # go 编译命令，会在当前目录下生成gogs
```

以上命令没有出错代表一切正常，同时我们开始准备`Dockerfile`，参考官方`Dockerfile.aarch64`，如下

##### 官方`Dockerfile`

```shell
FROM aarch64/alpine:3.5

# Install system utils & Gogs runtime dependencies
ADD https://github.com/tianon/gosu/releases/download/1.9/gosu-arm64 /usr/sbin/gosu
RUN chmod +x /usr/sbin/gosu \
  && echo http://dl-2.alpinelinux.org/alpine/edge/community/ >> /etc/apk/repositories \
  && apk --no-cache --no-progress add \
    bash \
    ca-certificates \
    curl \
    git \
    linux-pam \
    openssh \
    s6 \
    shadow \
    socat \
    tzdata

ENV GOGS_CUSTOM /data/gogs

# Configure LibC Name Service
COPY docker/nsswitch.conf /etc/nsswitch.conf
COPY docker /app/gogs/docker
COPY templates /app/gogs/templates
COPY public /app/gogs/public

WORKDIR /app/gogs/build
COPY . .

RUN    ./docker/build-go.sh \
    && ./docker/build.sh \
    && ./docker/finalize.sh

# Configure Docker Container
VOLUME ["/data"]
EXPOSE 22 3000
ENTRYPOINT ["/app/gogs/docker/start.sh"]
CMD ["/bin/s6-svscan", "/app/gogs/docker/s6/"]
```

但是，该文件的构建过程容易出错，所以我们可以删减编译步骤，从上面文件可以看出，`gogs`启动需要三个目录`templates`、`public`以及`conf`。将我们需要的文件从容器中拷贝出来，编写简化的`Dockerfile`，如下所示

##### 精简`Dockerfile`

```shell
[root@drone-ci gogs]# l
total 26780
drwxr-xr-x  7 root root     4096 Mar 25 04:45 conf
-rw-r--r--  1 root root      231 Mar 25 05:23 Dockerfile
-rwxr-xr-x  1 root root 27405032 Mar 25 04:49 gogs
drwxr-xr-x  8 root root     4096 Mar 25 04:45 public
drwxr-xr-x 11 root root     4096 Mar 25 04:45 templates
[root@drone-ci gogs]# cat Dockerfile
FROM arm64v8/alpine
RUN mkdir /app/antilope -p
ENV PATH /app/antilope:$PATH
COPY gogs /app/antilope/gogs
COPY public /app/antilope/public
COPY templates /app/antilope/templates

VOLUME ["/data"]

EXPOSE 22 3000
CMD ["gogs", "web"]
```

最后，运行我们构建好的`gogs`镜像，打开浏览器访问我们应用的地址，就可以看到安装界面。

> **Note：安装完数据库才可以使用`gogs`**

以上就是在树莓派中源码编译安装`gogs`，如有错误或者建议，请留言反馈给我们。