### 概述

这是关于`kubenetes`的使用过程中问题的记录以及解决方法。

[TOC]

### 无法`ping`同节点的容器地址

问题现象是部署在一个节点上的容器，在其他机器上无法访问，只有节点主机可以访问

```shell
[root@docker21 deployment]# kubectl get pods -o wide
NAME                               READY     STATUS    RESTARTS   AGE       IP             NODE
hello-777896485d-7hnkp             1/1       Running   0          20h       10.244.2.39    docker20
nginx-deployment-9985f488c-78g2l   1/1       Running   0          12m       10.244.7.3     master
nginx-deployment-9985f488c-sxgkj   1/1       Running   0          12m       10.244.2.161   docker20
nginx-deployment-9985f488c-zk9j8   1/1       Running   0          12m       10.244.7.2     master
[root@docker21 deployment]# ping 10.244.2.161
PING 10.244.2.161 (10.244.2.161) 56(84) bytes of data.
64 bytes from 10.244.2.161: icmp_seq=1 ttl=63 time=0.479 ms
^C
--- 10.244.2.161 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.479/0.479/0.479/0.000 ms
[root@docker21 deployment]# ping 10.244.7.2
PING 10.244.7.2 (10.244.7.2) 56(84) bytes of data.
From 10.244.7.0 icmp_seq=1 Destination Host Prohibited=
```

通过在两个节点上查看路由表，异常的节点路由表，如下

```shell
Chain DOCKER-ISOLATION (1 references)
target     prot opt source               destination
DROP       all  --  0.0.0.0/0            0.0.0.0/0
DROP       all  --  0.0.0.0/0            0.0.0.0/0
DROP       all  --  0.0.0.0/0            0.0.0.0/0
DROP       all  --  0.0.0.0/0            0.0.0.0/0
DROP       all  --  0.0.0.0/0            0.0.0.0/0
DROP       all  --  0.0.0.0/0            0.0.0.0/0
DROP       all  --  0.0.0.0/0            0.0.0.0/0
DROP       all  --  0.0.0.0/0            0.0.0.0/0
DROP       all  --  0.0.0.0/0            0.0.0.0/0
DROP       all  --  0.0.0.0/0            0.0.0.0/0
DROP       all  --  0.0.0.0/0            0.0.0.0/0
DROP       all  --  0.0.0.0/0            0.0.0.0/0
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
```

正常的节点路由表

```shell
Chain DOCKER-ISOLATION (1 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
```

然后，尝试清空该主机上的所有路由表、删除node、删除网卡，如下所示

```shell
(env) [root@master docker]# iptables -F &&  iptables -X &&  iptables -F -t nat &&  iptables -X -t nat
ip link delete flannel.1 
ip link delete cni0
rm -rf /run/flannel/
rm -rf /etc/cni/net.d/10-flannel.conflist
```

最后，重新部署容器到该节点上，可以访问。

```shell
[root@docker21 deployment]# kubectl get pods  -o wide
NAME                               READY     STATUS    RESTARTS   AGE       IP             NODE
hello-777896485d-7hnkp             1/1       Running   0          21h       10.244.2.39    docker20
nginx-deployment-9985f488c-4vn7w   1/1       Running   0          51s       10.244.10.5    master
nginx-deployment-9985f488c-6j2wc   1/1       Running   0          3m        10.244.2.168   docker20
nginx-deployment-9985f488c-d56mc   1/1       Running   0          3m        10.244.2.169   docker20
nginx-deployment-9985f488c-lz52k   1/1       Running   0          6m        10.244.2.166   docker20
nginx-deployment-9985f488c-nblsd   1/1       Running   0          6m        10.244.2.165   docker20
nginx-deployment-9985f488c-phr6s   1/1       Running   0          51s       10.244.10.3    master
nginx-deployment-9985f488c-pwd2c   1/1       Running   0          3m        10.244.2.167   docker20
nginx-deployment-9985f488c-w5pk5   1/1       Running   0          51s       10.244.10.2    master
nginx-deployment-9985f488c-xgcxj   1/1       Running   0          51s       10.244.10.4    master
nginx-deployment-9985f488c-xp7zr   1/1       Running   0          6m        10.244.2.164   docker20
[root@docker21 deployment]# ping 10.244.10.4
PING 10.244.10.4 (10.244.10.4) 56(84) bytes of data.
64 bytes from 10.244.10.4: icmp_seq=1 ttl=63 time=0.948 ms
```

扩展性思考：如何监控集群中节点的路由变化，以及如何定制化路由表设置？

### 服务扩容到大数量时会出现`CreateContainerError`

等待一段时间过后，服务正常运行，待确认问题的实质，etcd性能、服务器性能、网络以及`kubernetes`本身的调度问题。

### `docker pull`镜像报错timeout

具体日志

```shell
Error response from daemon: Get https://registry-1.docker.io/v2/library/traefik/manifests/latest: Get https://auth.docker.io/token?scope=repository%3Alibrary%2Ftraefik%3Apull&service=registry.docker.io: dial tcp: lookup auth.docker.io on [::1]:53: read udp [::1]:47329->[::1]:53: read: connection refused
```

在本地`ping`该域名无法获取地址，修改本地的`DNS`服务器地址为google的`8.8.8.8`后还是报错，通过`hosts`文件添加对应域名的服务器地址。

```shell
[root@master ~]# dig @8.8.8.8 auth.docker.io

; <<>> DiG 9.9.4-RedHat-9.9.4-18.el7_1.3 <<>> @8.8.8.8 auth.docker.io
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41386
;; flags: qr rd ra; QUERY: 1, ANSWER: 8, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;auth.docker.io.                        IN      A

;; ANSWER SECTION:
auth.docker.io.         46      IN      A       54.152.209.167
auth.docker.io.         46      IN      A       35.169.231.249
auth.docker.io.         46      IN      A       34.200.90.16
auth.docker.io.         46      IN      A       34.200.28.105
auth.docker.io.         46      IN      A       52.22.181.254
auth.docker.io.         46      IN      A       52.54.216.153
auth.docker.io.         46      IN      A       52.204.202.231
auth.docker.io.         46      IN      A       54.164.230.151
```

```shell
[root@master etc]# cat /etc/hosts
52.204.202.231 registry-1.docker.io
54.152.209.167 auth.docker.io
```

之后，再次拉去镜像成功，待确认具体问题，猜想是`docker`启动参数设置有问题。

### keepalived脑裂

测试环境中由于三台`keepalived`机器中处于不同网段，在路由异常后导致集群脑裂，后续参考其他解决方案。

### kube-router重启

```shell
Events:
  Type     Reason     Age                From            Message
  ----     ------     ----               ----            -------
  Normal   Pulled     13m (x4 over 5h)   kubelet, node1  Container image "dhub.juxinli.com/cloudnativelabs/kube-router" already present on machine
  Normal   Created    13m (x4 over 5h)   kubelet, node1  Created container
  Normal   Started    13m (x4 over 5h)   kubelet, node1  Started container
  Normal   Killing    13m (x4 over 14m)  kubelet, node1  Killing container with id docker://kube-router:Container failed liveness probe.. Container will be killed and recreated.
  Warning  BackOff    6m (x30 over 13m)  kubelet, node1  Back-off restarting failed container
  Warning  Unhealthy  1m (x25 over 59m)  kubelet, node1  Liveness probe failed: HTTP probe failed with statuscode: 500
```

在系统日志中查看到如下日志

```shell
[root@master2 log]# cat messages| grep kube-router-4zmvf
May 30 17:38:01 harbortest etcd: read-only range request "key:\"/registry/events/kube-system/kube-router-4zmvf.153362921aa70f15\" " took too long (186.374578ms) to execute
May 30 17:53:03 harbortest etcd: read-only range request "key:\"/registry/events/kube-system/kube-router-4zmvf.153362921aa70f15\" " took too long (5.248197799s) to execute
May 30 18:23:04 harbortest etcd: read-only range request "key:\"/registry/events/kube-system/kube-router-4zmvf.153362921aa70f15\" " took too long (6.475365013s) to execute
May 30 18:23:05 harbortest etcd: read-only range request "key:\"/registry/events/kube-system/kube-router-4zmvf.153362921aa70f15\" " took too long (167.811417ms) to execute
May 30 18:23:14 harbortest etcd: read-only range request "key:\"/registry/pods/kube-system/kube-router-4zmvf\" " took too long (1.728088221s) to execute
May 30 18:23:14 harbortest etcd: read-only range request "key:\"/registry/events/kube-system/kube-router-4zmvf.153355310626cc4e\" " took too long (5.427772716s) to execute
May 30 18:23:26 harbortest etcd: read-only range request "key:\"/registry/events/kube-system/kube-router-4zmvf.153362921aa70f15\" " took too long (1.560690067s) to execute
May 30 18:23:29 harbortest etcd: read-only range request "key:\"/registry/events/kube-system/kube-router-4zmvf.15336508a04e0099\" " took too long (605.249675ms) to execute
May 30 18:23:31 harbortest etcd: read-only range request "key:\"/registry/events/kube-system/kube-router-4zmvf.15335530dbc7cdb0\" " took too long (264.588997ms) to execute
May 30 18:23:50 harbortest etcd: read-only range request "key:\"/registry/events/kube-system/kube-router-4zmvf.15336508a04e0099\" " took too long (357.364316ms) to execute
May 30 18:23:51 harbortest etcd: read-only range request "key:\"/registry/events/kube-system/kube-router-4zmvf.15335530dbc7cdb0\" " took too long (129.809979ms) to execute
May 30 18:24:04 harbortest etcd: read-only range request "key:\"/registry/events/kube-system/kube-router-4zmvf.153362921aa70f15\" " took too long (347.076464ms) to execute
May 30 18:24:09 harbortest etcd: read-only range request "key:\"/registry/pods/kube-system/kube-router-4zmvf\" " took too long (243.91226ms) to execute
May 30 18:25:09 harbortest etcd: read-only range request "key:\"/registry/pods/kube-system/kube-router-4zmvf\" " took too long (103.960511ms) to execute
May 30 18:25:23 harbortest etcd: read-only range request "key:\"/registry/pods/kube-system/kube-router-4zmvf\" " took too long (122.515363ms) to execute
May 30 18:26:56 harbortest etcd: read-only range request "key:\"/registry/pods/kube-system/kube-router-4zmvf\" " took too long (396.207439ms) to execute
```

不确定是不是`etcd`集群性能导致的问题。

### kube-router模式下清理节点

```shell
iptables -L -n
iptables -P INPUT ACCEPT
rm -f /etc/cni/net.d/*
ip link delete kube-bridge
```

### 系统内核恐慌

```shell
 kernel:skbuff: skb_under_panic: text:ffffffff8153ae35 len:130 put:14 head:ffff8800ba626a00 data:ffff8800ba6269ee tail:0x70 end:0xc0 dev:br0
```

系统报错，操作系统信息

```shell
[root@node2 ~]# uname -r
3.10.0-229.7.2.el7.x86_64
[root@node2 ~]# cat /proc/version
Linux version 3.10.0-229.7.2.el7.x86_64 (builder@kbuilder.dev.centos.org) (gcc version 4.8.3 20140911 (Red Hat 4.8.3-9) (GCC) ) #1 SMP Tue Jun 23 22:06:11 UTC 2015
[root@node2 ~]# cat /etc/issue
\S
Kernel \r on an \m

[root@node2 ~]# cat /etc/redhat-release
CentOS Linux release 7.1.1503 (Core)

[root@node1 kubernetes]# cat /proc/version
Linux version 3.10.0-229.7.2.el7.x86_64 (builder@kbuilder.dev.centos.org) (gcc version 4.8.3 20140911 (Red Hat 4.8.3-9) (GCC) ) #1 SMP Tue Jun 23 22:06:11 UTC 2015

```

同样有台机器报同样的错误，系统版本如下

```shell
[root@node2 exec]# cat /proc/version
Linux version 3.10.0-229.7.2.el7.x86_64 (builder@kbuilder.dev.centos.org) (gcc version 4.8.3 20140911 (Red Hat 4.8.3-9) (GCC) ) #1 SMP Tue Jun 23 22:06:11 UTC 2015
```

暂时没有出现错误的机器，系统版本如下

```shell
[root@node3 centos]# cat /proc/version
Linux version 3.10.0-693.21.1.el7.x86_64 (builder@kbuilder.dev.centos.org) (gcc version 4.8.5 20150623 (Red Hat 4.8.5-16) (GCC) ) #1 SMP Wed Mar 7 19:03:37 UTC 2018
```

查看`centos`官方的`bug`记录，找到一个类似报错记录https://bugs.centos.org/view.php?id=12636

不确定是否因为系统问题，内核不支持`kube-router`。

### 集群VIP访问失败

```shell
Error: Get https://10.96.0.1:443/api/v1/namespaces/kube-system/configmaps?labelSelector=OWNER%!D(MISSING)TILLER: dial tcp 10.96.0.1:443: i/o timeout
```

该集群是第二次部署，没有清楚路由，待重新部署后尝试。

```shell
[root@master2 jenkins]# iptables -L -n
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
KUBE-FIREWALL  all  --  0.0.0.0/0            0.0.0.0/0

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DOCKER-ISOLATION  all  --  0.0.0.0/0            0.0.0.0/0
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0
KUBE-FORWARD  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes forward rules */
ACCEPT     all  --  10.244.0.0/16        0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            10.244.0.0/16
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            /* allow outbound traffic from pods */
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            /* allow inbound traffic to pods */
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            /* allow outbound node port traffic on node interface with which node ip is associated */

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
KUBE-FIREWALL  all  --  0.0.0.0/0            0.0.0.0/0

Chain DOCKER (1 references)
target     prot opt source               destination

Chain DOCKER-ISOLATION (1 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0

Chain KUBE-FIREWALL (2 references)
target     prot opt source               destination
DROP       all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes firewall for dropping marked packets */ mark match 0x8000/0x8000

Chain KUBE-FORWARD (1 references)
target     prot opt source               destination
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes forwarding rules */ mark match 0x4000/0x4000
ACCEPT     all  --  10.244.0.0/16        0.0.0.0/0            /* kubernetes forwarding conntrack pod source rule */ ctstate RELATED,ESTABLISHED
ACCEPT     all  --  0.0.0.0/0            10.244.0.0/16        /* kubernetes forwarding conntrack pod destination rule */ ctstate RELATED,ESTABLISHED

Chain KUBE-SERVICES (2 references)
target     prot opt source               destination
```

问题已解决，针对在一台机器上多次部署，在每次部署前需要清理一下路由。

### BUG: soft lockup

```shell
 kernel:NMI watchdog: BUG: soft lockup - CPU#1 stuck for 21s! [khugepaged:32]
```

查看资料显示是系统负载过高，参考[lockup博客](https://blog.csdn.net/zgl07/article/details/46493421/)。

对于生产级别，硬件需要高质量以及做大量性能测试。

### kube-router显示的运行时间与集群查看的时间不一致

节点查看的时间

```shell
[root@node5 rook]# docker ps
doCONTAINER ID        IMAGE                                                                                                                                   COMMAND                  CREATED             STATUS              PORTS               NAMES
06e0c2dd3b4a        cloudnativelabs/kube-router@sha256:ef8dd135f24bd79c7810b905ebf80ebdd4d98f48c1167e4542967969154bdb2e                                     "/usr/local/bin/ku..."   33 minutes ago      Up 33 minutes                           k8s_kube-router_kube-router-l9hnb_kube-system_eefec836-6960-11e8-a0e1-525400bcf250_1
```

`master`节点查看的时间

```shell
kube-router-l9hnb                       1/1       Running   1          19h       192.168.100.90    node5
```

### 集群主机中的`pod`路由时间不稳定

```shell
[root@master2 demo]# ping 10.244.5.3
PING 10.244.5.3 (10.244.5.3) 56(84) bytes of data.
64 bytes from 10.244.5.3: icmp_seq=1 ttl=63 time=81.6 ms
64 bytes from 10.244.5.3: icmp_seq=2 ttl=63 time=0.978 ms
64 bytes from 10.244.5.3: icmp_seq=3 ttl=63 time=0.648 ms
64 bytes from 10.244.5.3: icmp_seq=4 ttl=63 time=1.62 ms
64 bytes from 10.244.5.3: icmp_seq=5 ttl=63 time=0.699 ms
64 bytes from 10.244.5.3: icmp_seq=6 ttl=63 time=0.674 ms
64 bytes from 10.244.5.3: icmp_seq=7 ttl=63 time=0.596 ms
64 bytes from 10.244.5.3: icmp_seq=8 ttl=63 time=38.0 ms
```

不确定原因是不是因为`iptables`维护路由表的性能，参考[iptables大规模应用的局限](https://linux.cn/article-8474-1.html?utm_source=rss&utm_medium=rss)。

### pod挂载pvc，一直处于`init`状态

通过`pods`查看到`node`主机，在日志中显示

```shell
Jun  7 05:18:45 node5 kubelet: E0607 05:18:45.045376    1988 file.go:76] Unable to read manifest path "/etc/kubernetes/manifests": path does not exist, ignoring
Jun  7 05:19:05 node5 kubelet: E0607 05:19:05.045939    1988 file.go:76] Unable to read manifest path "/etc/kubernetes/manifests": path does not exist, ignoring
Jun  7 05:19:25 node5 kubelet: E0607 05:19:25.046807    1988 file.go:76] Unable to read manifest path "/etc/kubernetes/manifests": path does not exist, ignoring
Jun  7 05:19:34 node5 kubelet: I0607 05:19:34.279531    1988 operation_generator.go:447] MountVolume.WaitForAttach entering for volume "pvc-284a9188-6a33-11e8-a0e1-525400bcf250" (UniqueName: "kubernetes.io/iscsi/10.96.94.155:3260:iqn.2016-09.com.openebs.jiva:pvc-284a9188-6a33-11e8-a0e1-525400bcf250:0") pod "solid-starfish-mysql-b8cd645b9-8stgg" (UID: "28837f50-6a33-11e8-a0e1-525400bcf250") DevicePath ""
Jun  7 05:19:34 node5 kubelet: E0607 05:19:34.279815    1988 iscsi_util.go:209] iscsi: could not read iface default error:
Jun  7 05:19:34 node5 kubelet: E0607 05:19:34.279923    1988 nestedpendingoperations.go:263] Operation for "\"kubernetes.io/iscsi/10.96.94.155:3260:iqn.2016-09.com.openebs.jiva:pvc-284a9188-6a33-11e8-a0e1-525400bcf250:0\"" failed. No retries permitted until 2018-06-07 05:21:36.279869237 -0400 EDT m=+11037.404454196 (durationBeforeRetry 2m2s). Error: "MountVolume.WaitForAttach failed for volume \"pvc-284a9188-6a33-11e8-a0e1-525400bcf250\" (UniqueName: \"kubernetes.io/iscsi/10.96.94.155:3260:iqn.2016-09.com.openebs.jiva:pvc-284a9188-6a33-11e8-a0e1-525400bcf250:0\") pod \"solid-starfish-mysql-b8cd645b9-8stgg\" (UID: \"28837f50-6a33-11e8-a0e1-525400bcf250\") : executable file not found in $PATH"
Jun  7 05:19:45 node5 kubelet: E0607 05:19:45.047326    1988 file.go:76] Unable to read manifest path "/etc/kubernetes/manifests": path does not exist, ignoring
```

根据日志，猜测`iscsi`没有安装，通过`systemctl status iscsi.service`发现没有安装，之后通过安装`iscsi`后，`pod`成功运行。

`openebs`的安装步骤见[官方文档](https://docs.openebs.io/docs/next/prerequisites.html#steps-for-configuring-and-verifying-open-iscsi)

### iscsi: failed to rescan session with error: iscsiadm: No session found.

