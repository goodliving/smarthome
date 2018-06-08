### 概述

这是关于`openebs`的存储卷安装过程记录，包括问题解决以及使用场景。

### 安装

对于`openebs`的安装参考[官方介绍](https://github.com/openebs/openebs/tree/master/k8s)，如下

```shell
kubectl apply -f openebs-operator.yaml
kubectl apply -f openebs-storageclasses.yaml
```

之后，安装一个应用做实例，以官方的`jenkins`为例，

```shell
kubectl apply -f jenkins.yml
```

最后，访问`jenkins`的地址来确认安装正确。

```shell
[root@master2 jenkins]# kubectl get svc
NAME                                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
jenkins-svc                                         NodePort    10.96.48.20      <none>        80:32186/TCP        10m
kubernetes                                          ClusterIP   10.96.0.1        <none>        443/TCP             1h
maya-apiserver-service                              ClusterIP   10.110.1.46      <none>        5656/TCP            26m
pvc-e7ab21fe-6963-11e8-a0e1-525400bcf250-ctrl-svc   ClusterIP   10.110.202.191   <none>        3260/TCP,9501/TCP   10m
[root@master2 jenkins]# kubectl get pods -o wide
NAME                                                             READY     STATUS    RESTARTS   AGE       IP            NODE
jenkins-6bc67f99d7-f7ddm                                         1/1       Running   0          10m       10.244.2.10   node1
maya-apiserver-54c9d75c5b-shlkw                                  1/1       Running   0          26m       10.244.4.2    node4
openebs-provisioner-8696d4b949-49rxq                             1/1       Running   0          26m       10.244.2.4    node1
pvc-e7ab21fe-6963-11e8-a0e1-525400bcf250-ctrl-78c88bf54d-p4kr6   2/2       Running   0          10m       10.244.2.7    node1
pvc-e7ab21fe-6963-11e8-a0e1-525400bcf250-rep-54d6fbb879-4jzwl    1/1       Running   0          10m       10.244.4.4    node4
pvc-e7ab21fe-6963-11e8-a0e1-525400bcf250-rep-54d6fbb879-b24q9    1/1       Running   0          10m       10.244.2.9    node1
pvc-e7ab21fe-6963-11e8-a0e1-525400bcf250-rep-54d6fbb879-d65b9    1/1       Running   0          10m       10.244.5.5    node5
```

### 备份

当一个有状态的容器，申请一个`pvc`，当`openebs`分配一个存储卷给它，当主机上的文件损坏，如何实现容灾备份。

### 存储主机

通过将节点标记为存储类，不将业务容器调度到该主机上。