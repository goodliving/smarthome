### 概述

这是关于在树莓派3中安装`ubuntu16.04`的arm64位系统，镜像系统是在[chainsx](https://github.com/chainsx/ubuntu64-rpi)中，以下记录在安装系统进行环境的配置，以便以后安装`docker`、`kubernets`等软件。

#### 关闭selinux

`ubuntu`系统默认不安装

#### `Sysctl`设置

```shell
[root@node1 net.d]# cat /etc/sysctl.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
root@master2:~# sysctl -p /etc/sysctl.conf
sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-ip6tables: No such file or directory
sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-iptables: No such file or directory
root@master2:~# modprobe br_netfilter
root@master2:~# ls /proc/sys/net/bridge
bridge-nf-call-arptables  bridge-nf-call-iptables        bridge-nf-filter-vlan-tagged
bridge-nf-call-ip6tables  bridge-nf-filter-pppoe-tagged  bridge-nf-pass-vlan-input-dev
[root@node1 net.d]# sysctl -p /etc/sysctl.conf # 出现以下表示已经生效
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

#### 添加apt-transport-https

```shell
root@ubuntu:/etc/apt/sources.list.d# apt-get install apt-transport-https
```

#### 添加docker-ce阿里软件源公钥

```shell
root@ubuntu:/etc/apt/sources.list.d# gpg --keyserver keyserver.ubuntu.com --recv  7EA0A9C3F273FCD8
root@ubuntu:/etc/apt/sources.list.d# gpg --export --armor 7EA0A9C3F273FCD8| sudo apt-key add -
```

#### docker-ce软件源地址

```shell
root@master1:/etc/apt/sources.list.d# pwd
/etc/apt/sources.list.d
root@master1:/etc/apt/sources.list.d# cat docker-ce.list
deb https://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial edge
```

#### Segmentation fault处理方法

在`apt-get install`安装包时，报`Segmentation fault`，找到如下错误

```shell
Errors were encountered while processing:
 python3.5
 python3
 dh-python
```

然后，在`/var/lib/dpkg/info`后，删除这几个开头的相关文件名后，重新使用`apt-get install -f `下载即可。

#### 添加kubernets阿里软件源公钥

```shell
root@ubuntu:/etc/apt/sources.list.d# gpg --keyserver keyserver.ubuntu.com --recv  3746C208A7317B0F
root@ubuntu:/etc/apt/sources.list.d# gpg --export --armor 3746C208A7317B0F| sudo apt-key add -
```

#### kubernets软件源地址

```shell
root@master1:/etc/apt/sources.list.d# pwd
/etc/apt/sources.list.d
root@master1:/etc/apt/sources.list.d# cat kubernets.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
```

#### 修改linux时区为CST

```shell
[root@drone-ci ~]# ln -s /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime
```

#### 修改linux时区为UTC

```shell
[root@drone-ci ~]# ln -sf /usr/share/zoneinfo/UTC /etc/localtime
```

