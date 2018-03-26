### 概述

这是在树莓派`arm64`系统中安装`kubernets`高可用集群的过程记录。

### 模块

本次安装采用`kubeadm`命令，采用`keepalived`做高可用、外部的`etcd`服务以及0私有镜像仓库，如下所示

- [x] [keepalived](https://github.com/goodliving/smarthome/tree/master/kubernets/keepalived)
- [x] [etcd](https://github.com/goodliving/smarthome/tree/master/kubernets/etcd)
- [x] [registry](https://github.com/goodliving/smarthome/tree/master/kubernets/gitOps/registry)
- [x] [kubeadm](https://github.com/goodliving/smarthome/tree/master/kubernets/kubeadm)

具体安装详情参考以上几个模块。

### Todos

为了加深对于`kubernets`的理解，现计划翻译官方英文使用教程，有以下部分

- [ ] #### [Web UI (Dashboard)](https://github.com/goodliving/smarthome/tree/master/kubernets/learn/webui)


- [ ] #### Using the kubectl Command-line

- [ ] #### Configuring Pods and Containers

- [ ] #### Running Applications

- [ ] #### Running Jobs

- [ ] #### Accessing Applications in a Cluster

- [ ] #### Monitoring, Logging, and Debugging

- [ ] #### Accessing the Kubernetes API

- [ ] #### Using TLS

- [ ] #### Administering a Cluster

- [ ] #### Administering Federation

- [ ] #### Managing Stateful Applications

- [ ] #### Cluster Daemons

- [ ] #### Managing GPUs

- [ ] #### Managing HugePages

