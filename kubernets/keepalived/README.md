### 概述

这是关于`kubernets`高可用安装`keepalived`过程描述，其中`keepalived`采用`rpm`包安装。

### 安装

以下是使用两台树莓派`centos`搭建高可用，环境信息如下

| 主机IP地址          | 子网掩码          | 角色     |
| --------------- | ------------- | ------ |
| 192.168.137.100 | 255.255.255.0 | master |
| 192.168.137.103 | 255.255.255.0 | backup |

> **网关地址为192.168.137.1/24，角色说明参考官方文档说明**

之后通过`yum install keepalived`安装即可，接下来需要对于配置文件`keepalived.conf`进行修改。

> **对于不能访问外网机器，可以通过下载源码编译来安装，在此不做详细介绍。**

master的`keepalived`配置文件，如下

```shell
[root@master keepalived]# pwd
/etc/keepalived
[root@master keepalived]# cat keepalived.conf
global_defs {
   router_id LVS_k8s
}

vrrp_script CheckK8sMaster {
    script "curl -k https://192.168.137.200:6443"
    interval 3
    timeout 9
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0 # 修改成对应网卡名称
    virtual_router_id 61
    # 主节点权重最高 依次减少
    priority 120
    advert_int 1
    #修改为本地IP
    mcast_src_ip 192.168.137.100
    nopreempt
    authentication {
        auth_type PASS
        auth_pass sqP05dQgMSlzrxHj
    }
    unicast_peer {
        #注释掉本地IP
        #192.168.137.100
         192.168.137.103
    }
    virtual_ipaddress {
        192.168.137.200/24
    }
    track_script {
        CheckK8sMaster
    }

}
```

根据注释修改成对应的值，在另一台机器上设置类似，修改对应`ip`。然后，启动两台机器上的`keepalived`

```shell
[root@master keepalived]# systemctl enable keepalived ; systemctl start keepalived
```

启动完毕，通过`systemctl status keepalived`来查看启动日志，确认无误。

##### master机器

```shell
[root@master keepalived]# systemctl status keepalived
● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2018-03-25 10:25:36 UTC; 32min ago
  Process: 900 ExecStart=/usr/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 901 (keepalived)
   Memory: 2.1M
   CGroup: /system.slice/keepalived.service
           ├─901 /usr/sbin/keepalived -D
           ├─902 /usr/sbin/keepalived -D
           └─903 /usr/sbin/keepalived -D

Mar 25 10:25:38 master Keepalived_vrrp[903]: Sending gratuitous ARP on eth0 for 192.168.137.200
Mar 25 10:25:38 master Keepalived_vrrp[903]: Sending gratuitous ARP on eth0 for 192.168.137.200
Mar 25 10:25:38 master Keepalived_vrrp[903]: Sending gratuitous ARP on eth0 for 192.168.137.200
Mar 25 10:25:38 master Keepalived_vrrp[903]: Sending gratuitous ARP on eth0 for 192.168.137.200
Mar 25 10:25:43 master Keepalived_vrrp[903]: Sending gratuitous ARP on eth0 for 192.168.137.200
Mar 25 10:25:43 master Keepalived_vrrp[903]: VRRP_Instance(VI_1) Sending/queueing gratuitous ARPs on eth0 for 192.168.137.200
Mar 25 10:25:43 master Keepalived_vrrp[903]: Sending gratuitous ARP on eth0 for 192.168.137.200
Mar 25 10:25:43 master Keepalived_vrrp[903]: Sending gratuitous ARP on eth0 for 192.168.137.200
Mar 25 10:25:43 master Keepalived_vrrp[903]: Sending gratuitous ARP on eth0 for 192.168.137.200
Mar 25 10:25:43 master Keepalived_vrrp[903]: Sending gratuitous ARP on eth0 for 192.168.137.200
```

##### backup机器

```shell
[root@node1 keepalived]# systemctl status keepalived
● keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2018-03-25 10:25:13 UTC; 33min ago
  Process: 9707 ExecStart=/usr/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 9708 (keepalived)
   Memory: 2.3M
   CGroup: /system.slice/keepalived.service
           ├─9708 /usr/sbin/keepalived -D
           ├─9709 /usr/sbin/keepalived -D
           └─9710 /usr/sbin/keepalived -D

Mar 25 10:25:33 node1 Keepalived_vrrp[9710]: VRRP_Instance(VI_1) Dropping received VRRP packet...
Mar 25 10:25:34 node1 Keepalived_vrrp[9710]: (VI_1): ip address associated with VRID 61 not present in MASTER advert : 192.168.137.200
Mar 25 10:25:34 node1 Keepalived_vrrp[9710]: bogus VRRP packet received on eth0 !!!
Mar 25 10:25:34 node1 Keepalived_vrrp[9710]: VRRP_Instance(VI_1) Dropping received VRRP packet...
Mar 25 10:25:35 node1 Keepalived_vrrp[9710]: (VI_1): ip address associated with VRID 61 not present in MASTER advert : 192.168.137.200
Mar 25 10:25:35 node1 Keepalived_vrrp[9710]: bogus VRRP packet received on eth0 !!!
Mar 25 10:25:35 node1 Keepalived_vrrp[9710]: VRRP_Instance(VI_1) Dropping received VRRP packet...
Mar 25 10:25:37 node1 Keepalived_vrrp[9710]: VRRP_Instance(VI_1) Received advert with higher priority 120, ours 110
Mar 25 10:25:37 node1 Keepalived_vrrp[9710]: VRRP_Instance(VI_1) Entering BACKUP STATE
Mar 25 10:25:37 node1 Keepalived_vrrp[9710]: VRRP_Instance(VI_1) removing protocol VIPs.
```

最后，我们在其他机器上通过`ping 192.168.137.200`来测试`keepalived`功能，还需要关闭`master`机器上的`keepalived`来确认`VIP`可以偏移。

> **Note：VIP的值不能被其他机器使用，同时该VIP要设置成能被访问，同一网段IP或者能路由到**

以上是`keeplived`的简单使用，对于其他高级使用参考官网文档。