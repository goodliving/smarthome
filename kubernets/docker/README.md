### 概述

这是在树莓派64位系统上安装`docker`的记录，包括问题处理等。

### 安装

* 软件镜像源
* 配置修改

##### 软件镜像源

在国内安装`docker`的话，默认软件源是从国外官网拉取安装，需要更换阿里云软件源地址，如下所示

```shell
[root@master1 yum.repos.d]# cat docker-ce.repo
[docker]
name= docker repo
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/aarch64/stable/
gpgcheck=0
```

之后，`yum makecache`拉取对应安装包地址，通过`yum install -y docker-ce`下载，正常结束后，`docker`就会安装在本机上，最后启动`systemctl enable docker;systemctl start docker`，查看版本来确认。

```shell
[root@master1 yum.repos.d]# docker version
Client:
 Version:       18.04.0-ce-dev
 API version:   1.37
 Go version:    go1.9.4
 Git commit:    8fabfd2
 Built: Fri Mar 16 14:45:59 2018
 OS/Arch:       linux/arm64
 Experimental:  false
 Orchestrator:  swarm

Server:
 Engine:
  Version:      18.04.0-ce-dev
  API version:  1.37 (minimum version 1.12)
  Go version:   go1.9.4
  Git commit:   8fabfd2
  Built:        Fri Mar 16 14:51:50 2018
  OS/Arch:      linux/arm64
  Experimental: false
```

##### 配置修改

默认配置文件目录在`/etc/docker`，在目录中新建`daemon.json`配置文件，如下所示

```shell
[root@master1 docker]# pwd
/etc/docker
[root@master1 docker]# cat daemon.json
{
  "registry-mirrors": ["https://h7f9z1uh.mirror.aliyuncs.com"], # 阿里源镜像加速
  "insecure-registries": ["192.168.137.62:5000"], # 私有镜像地址
  "storage-driver": "overlay2" # 存储方式为overlay2
}
```

以上配置好之后，`systemctl daemon-reload`来加载配置，之后`systemctl restart docker`重启即可。