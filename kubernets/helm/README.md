### 概述

这是关于`helm`的安装使用记录，包括问题描述、解决方案以及应用场景。

### 安装

首先从官方中将`helm`下载到本地，之后将可执行文件的路径放置到系统环境变量中，然后，我们就可以通过`helm init`进行默认安装，如下图

```shell
[root@docker21 helm]# helm init
Creating /root/.helm
Creating /root/.helm/repository
Creating /root/.helm/repository/cache
Creating /root/.helm/repository/local
Creating /root/.helm/plugins
Creating /root/.helm/starters
Creating /root/.helm/cache/archive
Creating /root/.helm/repository/repositories.yaml
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com
Adding local repo with URL: http://127.0.0.1:8879/charts
$HELM_HOME has been configured at /root/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!
```

需要注意的是，在初始化的过程中会安装`Tiller`，而这个会从谷歌的景象中下载对应的`Tiller`景象，国内无法直接访问，自行科学上网下载对应的镜像。

> **计划将`helm`7运行参数可配置，需要了解`helm`安装过程**

报错如下

```shell
[root@docker21 ~]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                           READY     STATUS             RESTARTS   AGE
default       hello-777896485d-7hnkp                         1/1       Running            0          46m
default       nginx-deployment-9985f488c-5m8vw               1/1       Running            0          1h
default       nginx-deployment-9985f488c-gjs97               1/1       Running            0          1h
default       nginx-deployment-9985f488c-gxqtt               1/1       Running            0          1h
development   nginx-test                                     1/1       Running            0          1d
kube-system   coredns-65dcdb4cf-pflnj                        1/1       Running            0          1d
kube-system   kube-apiserver-docker21                        1/1       Running            0          1d
kube-system   kube-controller-manager-docker21               1/1       Running            0          1d
kube-system   kube-flannel-ds-hv255                          1/1       Running            1          1d
kube-system   kube-flannel-ds-j95mq                          1/1       Running            0          1d
kube-system   kube-proxy-8pn8z                               1/1       Running            0          1d
kube-system   kube-proxy-dkjkj                               1/1       Running            0          1d
kube-system   kube-scheduler-docker21                        1/1       Running            0          1d
kube-system   tiller-deploy-7ccf99cd64-hftj6                 0/1       ImagePullBackOff   0          7m
kubeless      kubeless-controller-manager-5c585f7c88-rvsbz   1/1       Running            0          5h
```

通过查看该`pod`会出现如下错误

```shell
Events:
  Type     Reason                 Age              From               Message
  ----     ------                 ----             ----               -------
  Normal   Scheduled              8m               default-scheduler  Successfully assigned tiller-deploy-7ccf99cd64-hftj6 to docker20
  Normal   SuccessfulMountVolume  8m (x2 over 8m)  kubelet, docker20  MountVolume.SetUp succeeded for volume "default-token-pkjvp"
  Warning  Failed                 7m               kubelet, docker20  Failed to pull image "gcr.io/kubernetes-helm/tiller:v2.9.1": rpc error: code = Unknown desc = Error response from daemon: Get https://gcr.io/v1/_ping: dial tcp 64.233.189.82:443: i/o timeout
  Normal   SandboxChanged         7m               kubelet, docker20  Pod sandbox changed, it will be killed and re-created.
  Warning  Failed                 6m               kubelet, docker20  Failed to pull image "gcr.io/kubernetes-helm/tiller:v2.9.1": rpc error: code = Unknown desc = Error response from daemon: Get https://gcr.io/v1/_ping: dial tcp 108.177.125.82:443: i/o timeout
  Normal   BackOff                4m (x5 over 6m)  kubelet, docker20  Back-off pulling image "gcr.io/kubernetes-helm/tiller:v2.9.1"
  Normal   Pulling                4m (x4 over 7m)  kubelet, docker20  pulling image "gcr.io/kubernetes-helm/tiller:v2.9.1"
  Warning  Failed                 3m (x4 over 7m)  kubelet, docker20  Error: ErrImagePull
  Warning  Failed                 3m (x2 over 5m)  kubelet, docker20  Failed to pull image "gcr.io/kubernetes-helm/tiller:v2.9.1": rpc error: code = Unknown desc = Error response from daemon: Get https://gcr.io/v1/_ping: dial tcp 64.233.188.82:443: i/o timeout
  Warning  Failed                 2m (x8 over 6m)  kubelet, docker20  Error: ImagePullBackOff
```

之后将镜像下载到运行的`node`后，再次查看会发现已经在运行

```shell
tiller-deploy-7ccf99cd64-hftj6     1/1       Running   0          18m       10.244.2.41      docker20
```

### 使用

通过`helm install stable/mysql`为例来描述应用场景，当我们运行该命令时，会出现如下错误

```shell
[root@docker21 ~]# helm install stable/mysql
Error: no available release name found
```

博客文章[helm install报错](https://blog.csdn.net/guizaijianchic/article/details/80283624)上具体解决方法，当我们在环境中运行命令

```shell
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

之后，我们再次运行`helm install stable/mysql`后没有报错，正常下载，如下所示

```shell
[root@docker21 ~]# helm install stable/mysql
NAME:   limping-blackbird
LAST DEPLOYED: Thu May 24 16:25:58 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME                     TYPE    DATA  AGE
limping-blackbird-mysql  Opaque  2     0s

==> v1/ConfigMap
NAME                          DATA  AGE
limping-blackbird-mysql-test  1     0s

==> v1/PersistentVolumeClaim
NAME                     STATUS   VOLUME  CAPACITY  ACCESS MODES  STORAGECLASS  AGE
limping-blackbird-mysql  Pending  0s

==> v1/Service
NAME                     TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)   AGE
limping-blackbird-mysql  ClusterIP  10.107.87.101  <none>       3306/TCP  0s

==> v1beta1/Deployment
NAME                     DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
limping-blackbird-mysql  1        1        1           0          0s

==> v1/Pod(related)
NAME                                      READY  STATUS    RESTARTS  AGE
limping-blackbird-mysql-57bf96f5f8-5grdn  0/1    Init:0/1  0         0s


NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
limping-blackbird-mysql.default.svc.cluster.local

To get your root password run:

    MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default limping-blackbird-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:
    $ mysql -h limping-blackbird-mysql -p

To connect to your database directly from outside the K8s cluster:
    MYSQL_HOST=127.0.0.1
    MYSQL_PORT=3306

    # Execute the following commands to route the connection:
    export POD_NAME=$(kubectl get pods --namespace default -l "app=limping-blackbird-mysql" -o jsonpath="{.items[0].metadata.name}")
    kubectl port-forward $POD_NAME 3306:3306

    mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}
```

按照上述命令登陆`mysql`容器，出现如下错误

```shell
Events:
  Type     Reason                 Age               From               Message
  ----     ------                 ----              ----               -------
  Normal   Scheduled              20m               default-scheduler  Successfully assigned limping-blackbird-mysql-57bf96f5f8-5grdn to docker20
  Normal   SuccessfulMountVolume  20m               kubelet, docker20  MountVolume.SetUp succeeded for volume "default-token-wlrdh"
  Warning  FailedMount            2m (x8 over 18m)  kubelet, docker20  Unable to mount volumes for pod "limping-blackbird-mysql-57bf96f5f8-5grdn_default(166b9aed-5f2c-11e8-ad47-525400aa92f3)": timeout expired waiting for volumes to attach/mount for pod "default"/"limping-blackbird-mysql-57bf96f5f8-5grdn". list of unattached/unmounted volumes=[data]
```

> **在安装kubernetes时未安装`volume`，待`volumen`安装成功后再次尝试**

### 相关命令

| 命令             | 说明                      |
| -------------- | ----------------------- |
| helm ls        | 查看当前有哪些版本               |
| helm delete xx | 卸载xx版本                  |
| helm status xx | 查看xx版本的具体信息，部署的命名空间以及状态 |
| helm get -h    | 查看helm帮助手册              |

更具体的命令参考[helm命令介绍](https://docs.helm.sh/helm)。