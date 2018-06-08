### 概述

这是关于使用`kubelet`常用命令统计。

##### 查询集群节点

```shell
[root@master1 etcd]# kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
master1   Ready     master    1h        v1.9.6
master2   Ready     master    1h        v1.9.6
node1     Ready     <none>    1m        v1.9.6
node2     Ready     <none>    1h        v1.9.6
```

##### 查询集群所有命名空间的pod

```shell
[root@master1 etcd]# kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                              READY     STATUS    RESTARTS   AGE       IP                NODE
default       nginx-7587c6fdb6-nl79f            1/1       Running   0          1h        10.244.2.2        node2
kube-system   coredns-65dcdb4cf-6bh44           1/1       Running   0          1h        10.244.0.2        master1
kube-system   kube-apiserver-master1            1/1       Running   1          1h        192.168.137.100   master1
kube-system   kube-apiserver-master2            1/1       Running   0          1h        192.168.137.103   master2
kube-system   kube-controller-manager-master1   1/1       Running   0          1h        192.168.137.100   master1
kube-system   kube-controller-manager-master2   1/1       Running   0          1h        192.168.137.103   master2
kube-system   kube-flannel-ds-ch5pv             1/1       Running   0          1h        192.168.137.180   node2
kube-system   kube-flannel-ds-mkvr8             1/1       Running   0          1h        192.168.137.100   master1
kube-system   kube-flannel-ds-ngr4w             1/1       Running   0          5m        192.168.137.124   node1
kube-system   kube-flannel-ds-rh795             1/1       Running   0          1h        192.168.137.103   master2
kube-system   kube-proxy-5rv8p                  1/1       Running   0          5m        192.168.137.124   node1
kube-system   kube-proxy-65xhl                  1/1       Running   0          1h        192.168.137.100   master1
kube-system   kube-proxy-lrb77                  1/1       Running   0          1h        192.168.137.180   node2
kube-system   kube-proxy-vw4xh                  1/1       Running   0          1h        192.168.137.103   master2
kube-system   kube-scheduler-master1            1/1       Running   0          1h        192.168.137.100   master1
kube-system   kube-scheduler-master2            1/1       Running   0          1h        192.168.137.103   master2 
```

##### 创建service

```shell
[root@master1 opt]# kubectl run nginx --image=nginx --port=80
```

##### 查询默认命名空间pods

```shell
[root@master1 opt]# kubectl get pods -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-7587c6fdb6-nl79f   1/1       Running   0          50s       10.244.2.2   node2
```

##### 查询services

```shell
[root@master1 etcd]# kubectl get svc nginx -o wide
NAME      TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE       SELECTOR
nginx     NodePort   10.103.130.130   <none>        80:31105/TCP   1h        run=nginx
```

##### 删除pod

```shell
[root@master1 etcd]# kubectl delete deploy nginx
deployment "nginx" deleted
[root@master1 etcd]# kubectl get pods
No resources found.
```

##### 删除service

```shell
[root@master1 dashboard]# kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        2h
nginx        NodePort    10.103.130.130   <none>        80:31105/TCP   1h
[root@master1 dashboard]#
[root@master1 dashboard]# kubectl delete service nginx
service "nginx" deleted
[root@master1 dashboard]# kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   2h
```

##### 编辑应用yaml

```shell
[root@master1 dashbord]# kubectl -n kube-system edit service kubernetes-dashboard
```

##### dashbord访问

```shell
[root@master1 dashbord]# kubectl proxy --address='192.168.137.100' --port=8086 --accept-hosts='^*$' # 指定host，不然会出现无认证
```

##### exec命令

shell-demo.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shell-demo
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
```

```shell
kubectl create -f shell-demo.yaml
kubectl get pod shell-demo
[root@master1 ~]# kubectl exec -it shell-demo -- /bin/bash
root@shell-demo:/#
root@shell-demo:/# ls
bin   dev  home  media  opt   root  sbin  sys  usr
boot  etc  lib   mnt    proc  run   srv   tmp  var
```

##### master可部署

```shell
kubectl taint nodes --all node-role.kubernetes.io/master-
```

##### master清除可部署

```shell
kubectl cordon master1
```

##### 查询namespaces

```shell
root@master1:~# kubectl get namespaces
NAME          STATUS    AGE
default       Active    7h
kube-public   Active    7h
kube-system   Active    7h
```

##### 创建namespace

```shell
root@master1:~# cat namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
   name: development
   labels:
     name: development
root@master1:~# kubectl create -f namespace.yaml
namespace "development" created
root@master1:~# kubectl get namespaces
NAME          STATUS    AGE
default       Active    7h
development   Active    10s
kube-public   Active    7h
kube-system   Active    7h
```

##### 创建指定namespace应用

```shell
root@master1:~# cat test-namespace.yml
apiVersion: v1
kind: Pod
metadata:
   name: nginx-test
   labels:
     name: antilope
   namespace: development
spec:
  containers:
  - name: nginx-test
    image: 192.168.137.62:5000/antilope/nginx
    env:
    - name: NAME
      value: hello
    - name: MESSAGES
      value: world
    ports:
    - containerPort: 80
      hostPort: 80
root@master1:~# kubectl get pods -n development
NAME         READY     STATUS    RESTARTS   AGE
nginx-test   1/1       Running   0          19m
```

##### 查看集群组件状态

```shell
kubectl get componentstatuses	
```

##### 查看集群信息

```shell
kubectl cluster-info dump
```

