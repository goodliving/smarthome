### 概述

这是关于kubernets在树莓派centos7系统中安装v1.9.5版本的过程记录。

### 准备工作

* 环境配置
* 下载镜像
* 安装网络(weave)
* etcd安装
* k8s安装
* dashbord安装
* 加入集群节点
* 部署nginx应用

##### 环境配置



```shell
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token c69bc4.53d119e807ab34f8 192.168.137.100:6443 --discovery-token-ca-cert-hash sha256:d2819ed272eeec3ce0205a431d3a6379f8e27fdcb38fddf49ebca44c42a50ba8
```

> **需要注意的是**

### 问题处理

以下记录使用kubeadm安装过程中系统日志报错以及解决方法

##### ImagePullBackOff：Back-off pulling images 

在安装v1.9.5时，安装[官方安装指导](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file)中准备的google镜像，使用`kubeadm init --apiserver-advertise-address=192.168.137.100 --feature-gates=SelfHosting=true --kubernetes-version=v1.9.5`时，报如下错误

```shell
Mar 22 14:22:54 manager kubelet: E0322 14:22:54.678000    2217 pod_workers.go:186] Error syncing pod 413224efa82e36533ce93e30bd18e3a8 ("etcd-manager_kube-system(413224efa82e36533ce93e30bd18e3a8)"), skipping: failed to "StartContainer" for "etcd" with ImagePullBackOff: "Back-off pulling image \"gcr.io/google_containers/etcd-arm64:3.1.11\""
Mar 22 14:22:55 manager kubelet: E0322 14:22:54.873393    2217 kubelet_node_status.go:106] Unable to register node "manager" with API server: Post https://192.168.137.100:6443/api/v1/nodes: net/http: TLS handshake timeout
```

现发现官方记录的使用的etcd版本是`3.1.10`，日志中显示需要拉起`3.1.10`，将`etcd`版本改成对应的版本后，显示正常。

##### crictl not found in system path

在kubeadm初始化过程中，报如下错误

```shell
[root@manager ~]# kubeadm init --apiserver-advertise-address=192.168.137.100 --feature-gates=SelfHosting=true --kubernetes-version=v1.9.5
[init] Using Kubernetes version: v1.9.5
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks.
        [WARNING SystemVerification]: docker version is greater than the most recently validated version. Docker version: 18.04.0-ce-dev. Max validated version: 17.03
        [WARNING FileExisting-crictl]: crictl not found in system path
```

在`10-kubeadm.conf`文件添加`Environment="KUBELET_SWAP_ARGS=--fail-swap-on=false"`，如下所示

```shell
[root@manager kubelet.service.d]# pwd
/etc/systemd/system/kubelet.service.d
[root@manager kubelet.service.d]# cat 10-kubeadm.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_CADVISOR_ARGS=--cadvisor-port=0"
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"
Environment="KUBELET_CERTIFICATE_ARGS=--rotate-certificates=true --cert-dir=/var/lib/kubelet/pki"
Environment="KUBELET_SWAP_ARGS=--fail-swap-on=false" #这是添加的配置项
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CGROUP_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS
```

#####  cni config uninitialized

在kubeadmin启动后，报如下错误，是因为还没有安装网络模型

```shell
Mar 22 15:20:23 manager kubelet: W0322 15:20:23.519498    5679 cni.go:171] Unable to update cni config: No networks found in /etc/cni/net.d
Mar 22 15:20:23 manager kubelet: E0322 15:20:23.520304    5679 kubelet.go:2120] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```

以下是以安装`weave`网络模型为例，可以参考[官网](https://www.weave.works/docs/net/latest/kubernetes/)使用说明中安装此网络插件。

首先，创建好`/etc/cni/net.d`目录，之后下载`weave`可执行文件到本地，如下所示

```shell
[root@manager opt]# sudo curl -L git.io/weave -o /usr/local/bin/weave
[root@manager opt]# sudo chmod a+x /usr/local/bin/weave
[root@manager opt]# weave launch 192.168.137.100
```

等待脚本拉取镜像完毕后，显示如下

```shell
[root@manager net.d]# pwd
/etc/cni/net.d
[root@manager net.d]# l
total 4
-rw-r--r-- 1 root root 74 Mar 22 15:28 10-weave.conf
[root@manager net.d]# cat 10-weave.conf
{
    "name": "weave",
    "type": "weave-net",
    "hairpinMode": true
}
```

之后，日志文件中报错信息消除

```shell
Mar 22 15:28:14 manager kubelet: W0322 15:28:14.121472    5679 cni.go:171] Unable to update cni config: No networks found in /etc/cni/net.d
Mar 22 15:28:14 manager kubelet: E0322 15:28:14.122477    5679 kubelet.go:2120] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
Mar 22 15:28:14 manager dockerd: time="2018-03-22T15:28:14Z" level=info msg="shim reaped" id=4d5940191c64bb4c5fecc2795b61fddb4409b193e7bb5eadcdf131f490bec0ee module="containerd/tasks"
Mar 22 15:28:14 manager dockerd: time="2018-03-22T15:28:14.806534703Z" level=info msg="ignoring event" module=libcontainerd namespace=moby topic=/tasks/delete type="*events.TaskDelete"
Mar 22 15:28:28 manager dhclient[7761]: DHCPREQUEST on eth0 to 192.168.137.1 port 67 (xid=0x29bf3deb)
Mar 22 15:28:28 manager dhclient[7761]: DHCPACK from 192.168.137.1 (xid=0x29bf3deb)
Mar 22 15:28:28 manager NetworkManager[226]: <info>  [1521732508.3093] dhcp4 (eth0):   address 192.168.137.100
Mar 22 15:28:28 manager NetworkManager[226]: <info>  [1521732508.3094] dhcp4 (eth0):   plen 24 (255.255.255.0)
Mar 22 15:28:28 manager NetworkManager[226]: <info>  [1521732508.3095] dhcp4 (eth0):   gateway 192.168.137.1
Mar 22 15:28:28 manager NetworkManager[226]: <info>  [1521732508.3096] dhcp4 (eth0):   lease time 604800
Mar 22 15:28:28 manager NetworkManager[226]: <info>  [1521732508.3097] dhcp4 (eth0):   nameserver '192.168.137.1'
Mar 22 15:28:28 manager NetworkManager[226]: <info>  [1521732508.3097] dhcp4 (eth0):   domain name 'mshome.net'
Mar 22 15:28:28 manager NetworkManager[226]: <info>  [1521732508.3098] dhcp4 (eth0): state changed bound -> bound
Mar 22 15:28:28 manager dbus[204]: [system] Activating via systemd: service name='org.freedesktop.nm_dispatcher' unit='dbus-or
```

### TODOS

- [ ] gitOps系统设计