

# Web UI (Dashboard)

`dashboard`是一个基于`kubernets`用户接口的`web`应用。我们可以在`dashboard`上部署应用容器到一个`kubernets`集群中，也可以实时观察到容器状态以及管理集群本身的一些固有的资源。同时我们可以通过`dashbord`查看部署在集群中的应用详情列表，也可以单独创建或者修改`kubernets`资源，如`Deployment`、`Jobs`和`DaemonSets`等。例如，通过`dashbord`上面的部署指导步骤，我们可以创建一个`Deployment`，应用的弹性伸缩，重新一个`pod`等。

`dashboard`同时还提供一些`kubernets`集群状态详情，以及日志查看。

### 部署`Dashboard UI`

在集群安装过程中不会默认安装`dashboard`，需要我们手动安装，通过下面命令直接安装

```shell
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

以上部署默认是在`amd64`系统，下面介绍在`arm64`系统中安装`dashboard`，可以在[官方dashboard](https://github.com/kubernetes/dashboard/)中查看安装步骤，以下记录安装过程中问题解决方法，默认已经安装好一个集群，整个安装步骤有三个部分

* `kubernetes-dashboard-arm.yaml`编写
* 账户权限获取
* `heapster`安装

##### kubernetes-dashboard-arm.yaml

部署文件可以参考[dashboard官方项目路径](https://github.com/kubernetes/dashboard/tree/master/src/deploy/recommended)，其中需要注意的是将所有指定`amd`架构改成`arm`，以及添加`          - --heapster-host=http://heapster`参数，同时自行下载好镜像到本地，默认是从谷歌镜像仓库中下载。之后通过`kubelet create -f `安装即可。

> **不添加heapster参数，日志中会显示找不到`heapster`应用，参考[bug说明](https://github.com/kubernetes/dashboard/issues/1602)** 

##### 账户权限获取

在登录`dashboard`时，需要我们创建好账号，可以参考[账号部署指南](https://github.com/kubernetes/dashboard/wiki/Creating-sample-user)，主要是有两个账号文件

```shell
root@master1:~/account# l
total 8
-rw-r--r-- 1 root root 261 Mar 31 08:42 admin-role.yml
-rw-r--r-- 1 root root  90 Mar 31 06:44 admin-use.yml
root@master1:~/account# cat admin-role.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
root@master1:~/account# cat admin-use.yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```

之后，通过`kubelet create`创建`admin-use`用户。

##### `heapster`安装

`dashboard`安装必须要安装`heapster`，来提供集群数据获取和图形展示功能。整个模块需要安装`grafana`、`influxdb`以及`heapster`，部署模板文件可以参考[模板yaml文件](https://github.com/kubernetes/heapster/tree/master/deploy/kube-config/influxdb)。

> **将模板yaml文件中`amd`改成`arm`，同时下载谷歌镜像到本地**

```shell
root@master1:~/heapster# l
total 12
-rw-r--r-- 1 root root 2303 Mar 31 07:30 grafana.yaml
-rw-r--r-- 1 root root 1127 Mar 31 07:33 heapster.yaml
-rw-r--r-- 1 root root  987 Mar 31 07:32 influxdb.yaml
```
需要在heapster.yml文件中将heapster用户设置为管理员用户，添加如下内容
```shell
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: heapster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: heapster
  namespace: kube-system
```
heapster为普通用户的话，在dashboard页面中没有对应的图形显示，查看heapster容器日志会看到报没有权限
以上部署完毕，通过`kubelet`查看，如下表示正常

```shell
root@master1:~/dashbord# kubectl get pods -n kube-system
NAME                                    READY     STATUS    RESTARTS   AGE
coredns-65dcdb4cf-znnkd                 1/1       Running   0          8h
heapster-5bf88d8668-d2rgv               1/1       Running   0          6h
kube-apiserver-master1                  1/1       Running   4          8h
kube-controller-manager-master1         1/1       Running   0          8h
kube-flannel-ds-nmzk7                   1/1       Running   0          8h
kube-proxy-qx59j                        1/1       Running   0          8h
kube-scheduler-master1                  1/1       Running   1          8h
kubernetes-dashboard-6967979989-rk2sl   1/1       Running   0          6h
monitoring-grafana-667c575bd8-z2vqp     1/1       Running   0          6h
monitoring-influxdb-7b689fc88-vstcl     1/1       Running   0          6h

root@master1:~/dashbord# kubectl get svc -n kube-system
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
heapster               ClusterIP   10.106.87.20    <none>        80/TCP          6h
kube-dns               ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP   8h
kubernetes-dashboard   NodePort    10.98.104.183   <none>        443:30064/TCP   6h
monitoring-grafana     ClusterIP   10.104.212.67   <none>        80/TCP          6h
monitoring-influxdb    ClusterIP   10.100.113.11   <none>        8086/TCP        6h
```

### 登录`Dashboard UI`

`UI`登录方式有很多种，可以通过`kubelet`命令或者`master apiserver`登录。

##### command line proxy

在maser机器上输入`kubelet proxy`，设置本地代理与`apiserver`通信，然后我们就可以访问地址`http://localhost:8001/ui`，同时也可以修改`IP`和`port`等，具体信息通过`kubelet proxy -h`来查看。

> **kubectl proxy --address='192.168.137.100' --port=8086 --accept-hosts='^*$' # 指定host，不然会出现无认证**

##### Master server

我们可以直接访问master机器上的`apiserver`进入`UI`界面，url地址为`https://<kubernetes-master>/ui`，这种方式未尝试，以下介绍通过修改`dashboard`映射端口来访问。

```shell
root@master1:~/dashbord# kubectl edit service kubernetes-dashboard -n kube-system
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: 2018-03-31T07:36:24Z
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
  resourceVersion: "11647"
  selfLink: /api/v1/namespaces/kube-system/services/kubernetes-dashboard
  uid: 371526f6-34b6-11e8-8cd6-b827eb94d1f6
spec:
  clusterIP: 10.98.104.183
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 30064
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort # 将ClusterIP修改成NodePort，后通过nodeIP+port访问
status:
  loadBalancer: {}
```

然后打开`firefox`浏览器输入地址`https://master-server:port/ui`，最后认证方式有两种方式，以下介绍使用`token`验证。使用以前设置账号步骤中的用户来登录，获取其`token`方式如下

```shell
root@master1:~/dashbord# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')

Name:         admin-user-token-dpmfg
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=82d4abc1-34bf-11e8-8cd6-b827eb94d1f6

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLWRwbWZnIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI4MmQ0YWJjMS0zNGJmLTExZTgtOGNkNi1iODI3ZWI5NGQxZjYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.mzdas5zJVmsXtxSQujIcHWg3K41zW09vDwpZ9jSuATnv6-o2zH0oCVY7AodZGNE_gwg908NXPRIXqm7T3WsTMwPICCBuGm0zGBtc4hBjtnwnaw6nchW_LqQwC71sD3zD79yUvqpCq4ejrauG_BwULgptalYmEB1fE-TGIhRahTjLCvaA1OdbwlaoSQT_AiqSbqBzESu0Q8S8We5hcrM9ERkBLFAINiLwitpfBfRuO8m2NyeJZ5fXZwY0iqTPfosyfjDccb8p5fvGKkbaeI_C6pRQCtYc81vS-5NhEgJjpepHNvNiyafc45--jKNj0KJX3TO4KEWDkjHY7hsgPwaxfQ
```

输入获取的`token`登录即可进行入到详情页面中。

> **对于chrome浏览器登录会出现无法打开https页面，需要进行关闭https设置，自行查看相关设计教程**

