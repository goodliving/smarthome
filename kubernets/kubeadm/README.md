### 概述

这是关于在树莓派上使用`kubeadm`搭建个人的高可用`kubernets`集群。

### 安装

在使用`kubeadm`安装之前，需要配置好系统环境，包括`etcd`、`keepalived`以及`google`镜像。

##### 私有镜像仓库

个人镜像仓库搭建参考[私有registry的搭建](https://github.com/goodliving/smarthome/tree/master/kubernets/gitOps/registry)，之后在[阿里云构建服务](https://cr.console.aliyun.com/?spm=5176.1972343.0.2.3e0l87&accounttraceid=d5562594-ee5e-42b0-afbf-163079083830&accounttraceid=043dc657-40f4-47dc-9168-723176d72ef1#/imageList)中通过其海外构建选项来拉取`google`镜像，具体的操作请查看操作说明。

##### etcd安装

关于etcd的安装参考本项目下面的[etcd安装说明](https://github.com/goodliving/smarthome/tree/master/kubernets/etcd)。

##### keepalived安装

关于`keepalived`安装参考本项目下面的[keepalived安装说明](https://github.com/goodliving/smarthome/tree/master/kubernets/keepalived)。

##### kubeadm配置

以上软件安装好了之后，我们就可以安装`kebuadm`了，使用`yum install`直接下载，本次操作是下载`v1.9.6`版本，等下载完毕后，需要修改`kubelet`的一些配置项。

* `driver`类型
* 设置`bridge-nf-call-iptables`
* 本地镜像
* 安装`weave`网络

对于drive类型的修改，因为docker默认安装是`cgroupfs`，所以需要进行一些配置，如下

```shell
[root@master kubelet.service.d]# pwd
/etc/systemd/system/kubelet.service.d
[root@master kubelet.service.d]# cat 10-kubeadm.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_CADVISOR_ARGS=--cadvisor-port=0"
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd" # 需要修改成cgroupfs
Environment="KUBELET_CERTIFICATE_ARGS=--rotate-certificates=true --cert-dir=/var/lib/kubelet/pki"
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CGROUP_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS
```

> **Note：修改了系统服务配置项后需要使用`systemctl daemon-reload`来重新加载**

为了让`kubelet`启动的时候默认使用本地的镜像仓库地址，需要新增一个配置文件`20-pod-infra-image.conf`，内容如下

```shell
[root@master kubelet.service.d]# cat 20-pod-infra-image.conf
[Service]
Environment="KUBELET_EXTRA_ARGS=--pod-infra-container-image=192.168.137.62:5000/google_containers/pause-arm64:3.0" # 修改成自己的私有镜像仓库地址
```

在执行`kubeadm`过程中出现`/proc/sys/net/bridge/bridge-nf-call-iptables contents are not set to 1`，这个需要我们设置该参数，在以下文件添加下面两行

```shell
[root@node1 net.d]# cat /etc/sysctl.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
[root@node1 net.d]# sysctl -p /etc/sysctl.conf # 出现以下表示已经生效
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

以上准备完毕之后，我们就可以使用`kubeadm init --config /path/to/config.yml`来初始化安装`kubernets`集群，其中`config.yml`文件内容如下

```yaml
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
etcd:
  endpoints:
  - https://192.168.137.100:2379
  - https://192.168.137.103:2379
  - https://192.168.137.124:2379
  caFile: /etc/etcd/ssl/ca.pem
  certFile: /etc/etcd/ssl/etcd.pem
  keyFile: /etc/etcd/ssl/etcd-key.pem
  dataDir: /var/lib/etcd
networking:
  podSubnet: 10.244.0.0/16
kubernetesVersion: 1.9.6
api:
  advertiseAddress: "192.168.137.200"
token: "b99a00.a144ef80536d4344"
tokenTTL: "0s"
apiServerCertSANs:
- etcd-host1
- etcd-host2
- etcd-host3
- 192.168.137.100
- 192.168.137.103
- 192.168.137.124
- 192.168.137.200 # 虚IP地址
featureGates:
  CoreDNS: true # 使用coredns/coredns镜像，代替本身的DNS服务
imageRepository: "192.168.137.62:5000/google_containers" # 私有镜像地址
```

在`kubeadm init`成功之后，还需要安装网络模型，本次是选择安装`weave`，安装步骤比较简单，下载二进制文件到本地安装即可，如下所示

```shell
sudo curl -L git.io/weave -o /usr/local/bin/weave
sudo chmod a+x /usr/local/bin/weave
[root@node2 opt]# mkdir /etc/cni/net.d -p
[root@node2 opt]# ./weave setup 
```

上述执行完毕后，会在`/etc/cni/net.d`写入一个配置文件，之后在`master`机器上`kubectl get nodes`会出现节点状态是	`ready`表示正常。

```shell
[root@master1 dashboard]# kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
master1   Ready     master    23h       v1.9.6
```

这样，我们就在安装好了一台`master`节点，接下来是安装`keepalived`的另一个节点。首先在安装好的`master`机器上将`/etc/kubernets/pki`目录和配置文件`config.yml`拷贝到另一台机器对应的目录

```shell
[root@master1 dashboard]# scp -r /etc/kubernetes/pki root@192.168.137.103:/etc/kubernetes/
[root@master1 dashboard]# scp config.yml root@192.168.137.103:/etc/kubernetes/
```

然后，在另一节点上使用`kubeadm init --config /path/to/config.yml`即可，在这台机器上会使用刚才`master`生成的秘钥文件，等待结束后，这样就装好了高可用的`kubernets`集群，同样安装好`weave`网络插件，通过`kubelet get nodes`来查看集群运行状态

```shell
[root@master1 dashboard]# kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
master1   Ready     master    23h       v1.9.6
master2   Ready     master    1h        v1.9.6
```

这样，`master`机器安装完毕，然后我们添加节点机器到该集群中，使用刚才生成`kubeadm join`命令，在节点执行即可，同样需要安装网络`weave`，在此不做过多说明，最终结果如下

```shell
[root@master1 dashboard]# kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
master1   Ready     master    23h       v1.9.6
master2   Ready     master    1h        v1.9.6
node1     Ready     <none>    1h        v1.9.6
node2     Ready     <none>    1h        v1.9.6
```