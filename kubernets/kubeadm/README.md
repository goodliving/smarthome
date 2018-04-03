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

安装指定版本，通过`yum install kubeadm`来查看对应包名，然后修改成对应版本格式，如下

```shell
yum install kubelet-1.9.6-0.aarch64 kubeadm-1.9.6-0.aarch64 kubectl-1.9.6-0.aarch64 -y
```

* `driver`类型
* 设置`bridge-nf-call-iptables`
* 关闭`selinux`、`firewalld`
* 本地镜像
* 安装`flannel`网络

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

在安装过程中关闭`selinux`，有两种方法，永久修改是修改`/etc/selinux/config`配置文件，将`selinux`设置为`disabled`，之后重启服务器即可；临时修改是执行`setenforce 0 `即可，之后通过`/usr/sbin/sestatus -v`查看结果，如下

```shell
[root@master1 ~]# /usr/sbin/sestatus -v
SELinux status:                 disabled
```

关闭防火墙`firewall`是为了安装时添加对应的端口，执行以下命令

```shell
systemctl disable firewalld
systemctl stop firewalld
ip link delete flannel.1 
ip link delete cni0
iptables -L -n
# 如果没有重要规则，执行清空
iptables -P INPUT ACCEPT
rm -rf /run/flannel/
rm -rf /etc/cni/net.d/10-flannel.conflist
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
  podSubnet: 10.244.0.0/16 # 与kube-flannel.yml中组网保持一致，不然会无法运行报错
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

在`kubeadm init`成功之后，还需要安装网络模型，本次是选择安装`flannel`，安装步骤比较简单，从官网下载[kube-flannel.yml](https://github.com/coreos/flannel/tree/master/Documentation)，修改镜像架构`amd`为`arm`，之后安装即可

```shell
[root@master1 Documentation]# kubectl create -f kube-flannel.yml
```

> **在安装之前把镜像拉下来在本地，安装过程中可能会出现下载失败的情况**

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

然后，在另一节点上使用`kubeadm init --config /path/to/config.yml`即可，在这台机器上会使用刚才`master`生成的秘钥文件，等待结束后，这样就装好了高可用的`kubernets`集群，同样安装好`flannel`网络插件，通过`kubelet get nodes`来查看集群运行状态

```shell
[root@master1 dashboard]# kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
master1   Ready     master    23h       v1.9.6
master2   Ready     master    1h        v1.9.6
```

> **master2机器使用`kubectl create -f kube-flannel.yml --namespace=kube-system`，不指定命名空间会报已存在**

这样，`master`机器安装完毕，然后我们添加节点机器到该集群中，使用刚才生成`kubeadm join`命令，在节点执行即可，在此不做过多说明，最终结果如下

```shell
[root@master1 dashboard]# kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
master1   Ready     master    23h       v1.9.6
node1     Ready     <none>    1h        v1.9.6
node2     Ready     <none>    1h        v1.9.6
```
### 集群功能验证

接下我们通过在节点中运行一个`nginx`容器，通过访问其映射的端口来确认正常，首先创建一个`pod`

```shell
[root@master1 opt]# kubectl run nginx --image=nginx --port=80
deployment "nginx" created
[root@master1 opt]# kubectl get pods -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-7587c6fdb6-nl79f   1/1       Running   0          50s       10.244.2.2   node2
[root@master1 opt]#
[root@master1 opt]# kubectl describe pods/nginx-7587c6fdb6-nl79f
Name:           nginx-7587c6fdb6-nl79f
Namespace:      default
Node:           node2/192.168.137.180
Start Time:     Wed, 28 Mar 2018 14:23:49 +0000
Labels:         pod-template-hash=3143729862
                run=nginx
Annotations:    <none>
Status:         Running
IP:             10.244.2.2
Controlled By:  ReplicaSet/nginx-7587c6fdb6
Containers:
  nginx:
    Container ID:   docker://f80c72d5a0fd3fa11410d05b5849767b3d6162f3b097c9cee565c2d0aeeb6a16
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:c4ee0ecb376636258447e1d8effb56c09c75fe7acf756bf7c13efadf38aa0aca
    Port:           80/TCP
    State:          Running
      Started:      Wed, 28 Mar 2018 14:23:53 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-ljfzf (ro)
Conditions:
  Type           Status
  Initialized    True
  Ready          True
  PodScheduled   True
Volumes:
  default-token-ljfzf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-ljfzf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason                 Age   From               Message
  ----    ------                 ----  ----               -------
  Normal  Scheduled              23s   default-scheduler  Successfully assigned nginx-7587c6fdb6-nl79f to node2
  Normal  SuccessfulMountVolume  23s   kubelet, node2     MountVolume.SetUp succeeded for volume "default-token-ljfzf"
  Normal  Pulling                21s   kubelet, node2     pulling image "nginx"
  Normal  Pulled                 19s   kubelet, node2     Successfully pulled image "nginx"
  Normal  Created                19s   kubelet, node2     Created container
  Normal  Started                19s   kubelet, node2     Started container
```

可以看出，该`pod`已经获取到一个`ip`地址，但是直接访问其`ip`是无法访问的，可以通过将其端口映射到节点`node`的端口，如下

```shell
[root@master1 opt]# kubectl expose deploy nginx --type=NodePort --target-port=80 
service "nginx" exposed
[root@master1 opt]# kubectl get svc nginx
NAME      TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
nginx     NodePort   10.103.130.130   <none>        80:31105/TCP   13s
```

其中，`31105`是指节点主机的端口，通过`curl http://$nodeip:31105`来访问`pod`的`80`端口，如下

```shell
[root@node2 opt]# curl http://192.168.137.180:31105
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```



### 问题处理

以下记录使用kubeadm安装过程中系统日志报错以及解决方法

##### ImagePullBackOff：Back-off pulling images

该错误是因为`kubelet`拉取的镜像不存在，下载需要的镜像即可。

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

##### cni config uninitialized

在kubeadmin启动后，报如下错误，是因为还没有安装网络模型

```shell
Mar 22 15:20:23 manager kubelet: W0322 15:20:23.519498    5679 cni.go:171] Unable to update cni config: No networks found in /etc/cni/net.d
Mar 22 15:20:23 manager kubelet: E0322 15:20:23.520304    5679 kubelet.go:2120] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```

安装`fannel`网络插件即可，可以参考[官网](https://www.weave.works/docs/net/latest/kubernetes/)使用说明中安装此网络插件。

首先，创建好`/etc/cni/net.d`目录，之后通过`kubelet`安装即可，参考上面关于安装步骤。

##### flannel组网结构

在`config.yml`中`podSubnet`参数与`kube-flannel.yml`中`net-conf.json`组网信息需要保持一致，不一致的话添加节点会失败。

##### etcd日志报server is likely overloaded

不确定是树莓派系统性能影响的，待补充系统读写性能测试。