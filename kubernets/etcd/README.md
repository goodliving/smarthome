### 概述

这是关于`etcd`集群搭建过程的记录。

### 安装

我们在安装之前需要关闭`linux`防火墙以及安全加固，不关闭的话，在安装过程中日志汇报`dial tcp 10.1.2.172:2380: getsockopt: no route to host`。

```shell
[root@master sysconfig]# cat /etc/sysconfig/selinux

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled # 修改成disable，关闭加固，或者使用命令setenforce=0
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted

```

防火墙关闭使用以下命令，没有报错表示成功。

```shell
[root@master sysconfig]# service firewalld  stop
Redirecting to /bin/systemctl stop firewalld.service
[root@master sysconfig]# systemctl disable firewalld
```

当以前操作执行完毕后，接下来就正式安装`etcd`集群，主要有三个步骤

* `etcd`安装包
* `ssl`加密证书
* `etcd.service`系统服务

##### `etcd`安装包

从[官网下载页面](https://github.com/coreos/etcd/releases/)中选择版本下载，本次是以`v3.3.2`版本进行安装，将二进制文件拷贝到本地后放到`/usr/bin`目录中，多个主机同样的操作。

```shell
[root@master etcd]# mv etcd* /usr/bin/
```

##### `ssl`加密

为了保证通信安全，集群通信之间采用`TLS`加密，使用`cfssl`来生成证书和私钥。以下操作是参考[kubernets安装博客](https://www.kubernetes.org.cn/3536.html)完成。

首先创建`CA`配置文件

```shell
[root@master etcd]# cat ca-config.json
{
"signing": {
"default": {
  "expiry": "8760h"
},
"profiles": {
  "kubernetes": {
    "usages": [
        "signing",
        "key encipherment",
        "server auth",
        "client auth"
    ],
    "expiry": "8760h"
  }
}
}
}
```

```shell
[root@master etcd]# cat ca-csr.json
{
"CN": "kubernetes",
"key": {
"algo": "rsa",
"size": 2048
},
"names": [
{
  "C": "CN",
  "ST": "BeiJing",
  "L": "BeiJing",
  "O": "k8s",
  "OU": "System"
}
]
}
```

##### 生成`CA`证书和私钥

```shell
[root@master etcd]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca
[root@master etcd]# ls ca*
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```

##### 创建`etcd`证书签名请求

```shell
[root@master etcd]# cat etcd-csr.json
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.137.100", # 修改成自己的IP
    "192.168.137.103", # 修改成自己的IP
    "192.168.137.124" # 修改成自己的IP
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

以上需要注意的是集群`IP`修改成自己的。

##### 生成 etcd 证书和私钥

```shell
cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
[root@master etcd]# ls etcd*
etcd  etcd.csr  etcd-csr.json  etcdctl  etcd-key.pem  etcd.pem

[root@master etcd]# mkdir -p /etc/etcd/ssl
[root@master etcd]# cp etcd.pem etcd-key.pem  ca.pem /etc/etcd/ssl/
```

最后，将生成的`etcd.pem`、`etcd-key.pem`和`ca.pem`三个文件拷贝到集群所有主机的`/etc/etcd/ssl`目录中。

##### `etcd.service`系统服务

关于在centos中如何将`etcd`制作成系统服务，参考安装博客。以下记录在我的树梅派`arm64`系统中安装，对于`amd64`系统不做说明。`etcd.service`文件内容如下

```shell
[root@master etcd]# cat /etc/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
Environment=ETCD_UNSUPPORTED_ARCH=arm64 # 需要添加该环境变量在arm系统中运行
Environment=NODE_NAME=etcd-host1 # 节点名称
Environment=NODE_IP=192.168.137.100 # 修改成对应主机的IP
# Environment=NODE_IPS="192.168.137.100 192.168.137.103 192.168.137.124" # 不需要，可以删除
Environment=ETCD_NODES=etcd-host1=https://192.168.137.100:2380,etcd-host2=https://192.168.137.103:2380,etcd-host3=https://192.168.137.124:2380 # 集群节点信息
WorkingDirectory=/var/lib/etcd/ # 工作目录
ExecStart=/usr/bin/etcd \ # etcd脚本目录
        --name=${NODE_NAME} \
        --cert-file=/etc/etcd/ssl/etcd.pem \
        --key-file=/etc/etcd/ssl/etcd-key.pem \
        --peer-cert-file=/etc/etcd/ssl/etcd.pem \
        --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
        --trusted-ca-file=/etc/etcd/ssl/ca.pem \
        --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
        --initial-advertise-peer-urls=https://${NODE_IP}:2380 \
        --listen-peer-urls=https://${NODE_IP}:2380 \
        --listen-client-urls=https://${NODE_IP}:2379,http://127.0.0.1:2379 \
        --advertise-client-urls=https://${NODE_IP}:2379 \
        --initial-cluster-token=etcd-cluster-0 \
        --initial-cluster=${ETCD_NODES} \
        --initial-cluster-state=new \
        --data-dir=/var/lib/etcd/
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

将该`etcd.service`移动到`/etc/systemd/system/`系统服务目录中，在每个主机都设置完毕后，启动`etcd`服务。

```shell
[root@master etcd]# systemctl enable etcd
[root@master etcd]# systemctl start etcd
[root@master etcd]# journalctl -xe # 查看启动日志，根据报错日志修改配置项
```

最后，我们通过检查集群状态来验证安装成功，在每个主机下执行下面命令，当出现如下信息表示一切正常

```shell
[root@master etcd]#  etcdctl \
>   --endpoints=https://${NODE_IP}:2379  \ 
>   --ca-file=/etc/etcd/ssl/ca.pem \
>   --cert-file=/etc/etcd/ssl/etcd.pem \
>   --key-file=/etc/etcd/ssl/etcd-key.pem \
>   cluster-health
member 34872b39b0c7d050 is healthy: got healthy result from https://192.168.137.100:2379
member 725fb6e82d442a2d is healthy: got healthy result from https://192.168.137.124:2379
member 775c732a09ef19e8 is healthy: got healthy result from https://192.168.137.103:2379
cluster is healthy
```

这样，我们就有了一个三个节点的`etcd`集群，接下来有空会补充`etcd`的相关操作记录，包括添加、删除节点，集群备份，恢复等。

