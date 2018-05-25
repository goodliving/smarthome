### 概述

这是关于`kubenetes`的使用过程中问题的记录以及解决方法。

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