### 概述

这是关于在`kubernetes`中安装`kube-router`的过程记录，以及问题描述和解决方法。

### 安装

```shell
kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter.yaml
```

```shell
kubectl apply -f https://raw.githubusercontent.com/cloudnativelabs/kube-router/master/daemonset/kubeadm-kuberouter-all-features.yaml
```

```shell
kubectl -n kube-system delete ds kube-proxy
```

```shell
docker run --privileged --net=host dhub.juxinli.com/google_containers/kube-proxy-amd64:v1.9.6 kube-proxy --cleanup
```

> **修改kube-proxy-amd64的镜像为本地格式**



### dashboard

参考官方的[配置文件](https://github.com/cloudnativelabs/kube-router/blob/master/dashboard/kube-router.json)

### 清理

当需要重新安装集群时，需要清理机器上的环境，以下列出需要执行的步骤

##### `master`机器

```shell
kubeadm reset
ip link delete dummy0
ip link delete kube-dummy-if
ip link delete kube-bridge
ip link delete tunl0
iptables -F &&  iptables -X &&  iptables -F -t nat &&  iptables -X -t nat
iptables -L -n
iptables --flush  
iptables -tnat --flush
ifconfig tunl0 down
ip link delete tunl0
systemctl restart docker.service
```
##### 移除节点
```shell
kubectl drain node2 --delete-local-data --force --ignore-daemonsets
kubectl delete node node2
```
> **官方介绍重启`docker`可能避免了某些问题，参看[官方文档介绍](https://kubernetes.io/docs/setup/independent/troubleshooting-kubeadm/)**

##### node机器

```shell
kubeadm reset
ip link delete dummy0
ip link delete kube-dummy-if
ip link delete kube-bridge
ip link delete tunl0
iptables -F &&  iptables -X &&  iptables -F -t nat &&  iptables -X -t nat
iptables -L -n
iptables --flush  
iptables -tnat --flush
rm -rf /etc/kubernetes 
systemctl restart docker.service
```

> **/etc/kubernetes中有之前集群相关信息，需要删除**

### kubectl命令返回结果的时间长

使用`kubectl get pods`会出现返回不了的现象，是在安装`kubevirt.yaml`

查看`etcd`日志，发现如下日志

```shell
Jun  7 15:02:40 master1 etcd: read-only range request "key:\"/registry/ranges/serviceips\" " took too long (137.375276ms) to execute
Jun  7 15:02:41 master1 etcd: read-only range request "key:\"/registry/pods/kube-system/virt-api-6c984c49c6-t62g4\" " took too long (156.214897ms) to execute
Jun  7 15:02:41 master1 etcd: read-only range request "key:\"/registry/pods/kube-system/virt-api-6c984c49c6-x248n\" " took too long (149.897957ms) to execute
Jun  7 15:02:41 master1 etcd: read-only range request "key:\"/registry/pods/kube-system/virt-api-6c984c49c6-t62g4\" " took too long (103.511537ms) to execute
Jun  7 15:02:43 master1 etcd: rejected connection from "192.168.200.22:42964" (error "EOF", ServerName "")
Jun  7 15:02:45 master1 etcd: read-only range request "key:\"/registry/pods/kube-system/virt-controller-58c7584478-ss6kn\" " took too long (109.936838ms) to execute
Jun  7 15:02:45 master1 etcd: read-only range request "key:\"/registry/pods/kube-system/virt-controller-58c7584478-sl6wb\" " took too long (135.627672ms) to execute
Jun  7 15:02:45 master1 etcd: rejected connection from "192.168.200.22:42966" (error "EOF", ServerName "")
Jun  7 15:02:45 master1 etcd: read-only range request "key:\"/registry/pods/kube-system/virt-controller-58c7584478-sl6wb\" " took too long (118.654834ms) to execute
Jun  7 15:02:47 master1 etcd: read-only range request "key:\"/registry/apiregistration.k8s.io/apiservices/v1.batch\" " took too long (103.207455ms) to execute
Jun  7 15:02:47 master1 etcd: read-only range request "key:\"/registry/apiregistration.k8s.io/apiservices/v1alpha1.openebs.io\" " took too long (134.324867ms) to execute
Jun  7 15:02:47 master1 etcd: read-only range request "key:\"/registry/apiregistration.k8s.io/apiservices/v1alpha1.rook.io\" " took too long (171.126056ms) to execute
Jun  7 15:02:47 master1 etcd: read-only range request "key:\"/registry/apiregistration.k8s.io/apiservices/v1beta1.authorization.k8s.io\" " took too long (164.87084ms) to execute
Jun  7 15:02:47 master1 etcd: read-only range request "key:\"/registry/apiregistration.k8s.io/apiservices/v1alpha1.kubevirt.io\" " took too long (166.674527ms) to execute
Jun  7 15:02:47 master1 etcd: read-only range request "key:\"/registry/apiregistration.k8s.io/apiservices/v1beta1.policy\" " took too long (171.115153ms) to execute
Jun  7 15:02:48 master1 etcd: read-only range request "key:\"/registry/pods/kube-system/virt-handler-r4ck2\" " took too long (138.787488ms) to execute
Jun  7 15:02:48 master1 etcd: read-only range request "key:\"/registry/pods/kube-system/virt-handler-z7cl4\" " took too long (149.823925ms) to execute
Jun  7 15:02:48 master1 etcd: read-only range request "key:\"/registry/pods/kube-system/virt-handler-nfjj9\" " took too long (175.176259ms) to execute
Jun  7 15:02:48 master1 etcd: read-only range request "key:\"/registry/pods/kube-system/virt-handler-rjdfg\" " took too long (175.489005ms) to execute
Jun  7 15:02:48 master1 etcd: read-only range request "key:\"/registry/pods/kube-system/virt-handler-n66s9\" " took too long (214.81152ms) to execute
Jun  7 15:02:49 master1 etcd: read-only range request "key:\"/registry/pods/kube-system/virt-handler-r4ck2\" " took too long (208.082274ms) to execute
Jun  7 15:02:49 master1 etcd: read-only range request "key:\"/registry/pods/kube-system/virt-handler-z7cl4\" " took too long (174.446455ms) to execute
Jun  7 15:02:49 master1 etcd: read-only range request "key:\"/registry/pods/kube-system/virt-handler-rjdfg\" " took too long (140.029368ms) to execute
Jun  7 15:02:49 master1 etcd: read-only range request "key:\"/registry/pods/kube-system/virt-handler-nfjj9\" " took too long (164.008553ms) to execute
Jun  7 15:02:49 master1 etcd: read-only range request "key:\"/registry/services/specs/\" range_end:\"/registry/services/specs0\" " took too long (175.476507ms) to execute
```

###  sock_inode_cache报错，分配不到内存

```shell
Jun  8 17:26:20 node4 kernel: Memory cgroup stats for /kubepods/pod21198eae-6a36-11e8-a0e1-525400bcf250/c5e5ecf03a2ebe5dc06b7e8b28a59034d12a1ae6ce78e52220987ac561655c1d: cache:0KB rss:0KB rss_huge:0KB mapped_file:0KB swap:0KB inactive_anon:0KB active_anon:0KB inactive_file:0KB active_file:0KB unevictable:0KB
Jun  8 17:26:20 node4 kernel: Memory cgroup stats for /kubepods/pod21198eae-6a36-11e8-a0e1-525400bcf250/82d7805a09d66e34dafc753cd2dfa63c6d9fe6c4c907d73a887f42fda4208e2d: cache:0KB rss:0KB rss_huge:0KB mapped_file:0KB swap:0KB inactive_anon:0KB active_anon:0KB inactive_file:0KB active_file:0KB unevictable:0KB
Jun  8 17:26:20 node4 kernel: [ pid ]   uid  tgid total_vm      rss nr_ptes swapents oom_score_adj name
Jun  8 17:26:20 node4 kernel: [61659]     0 61659    10013      595      18        0          -999 runc:[2:INIT]
Jun  8 17:26:20 node4 kernel: Memory cgroup out of memory: Kill process 61662 (runc:[2:INIT]) score 0 or sacrifice child
Jun  8 17:26:20 node4 kernel: Killed process 61659 (runc:[2:INIT]) total-vm:40052kB, anon-rss:612kB, file-rss:1768kB, shmem-rss:0kB
Jun  8 17:26:20 node4 dockerd: time="2018-06-08T17:26:20.952834801+08:00" level=error msg="containerd: start container" error="oci runtime error: container_linux.go:247: starting container process caused \"process_linux.go:286: decoding sync type from init pipe caused \\\"read parent: connection reset by peer\\\"\"\n" id=82d7805a09d66e34dafc753cd2dfa63c6d9fe6c4c907d73a887f42fda4208e2d
Jun  8 17:26:20 node4 dockerd: time="2018-06-08T17:26:20.954564911+08:00" level=error msg="Create container failed with error: oci runtime error: container_linux.go:247: starting container process caused \"process_linux.go:286: decoding sync type from init pipe caused \\\"read parent: connection reset by peer\\\"\"\n"
Jun  8 17:26:20 node4 dockerd: time="2018-06-08T17:26:20.972300353+08:00" level=error msg="Handler for POST /v1.26/containers/82d7805a09d66e34dafc753cd2dfa63c6d9fe6c4c907d73a887f42fda4208e2d/start returned error: oci runtime error: container_linux.go:247: starting container process caused \"process_linux.go:286: decoding sync type from init pipe caused \\\"read parent: connection reset by peer\\\"\"\n"
Jun  8 17:26:20 node4 kubelet: W0608 17:26:20.974842    2241 container.go:393] Failed to create summary reader for "/kubepods/pod21198eae-6a36-11e8-a0e1-525400bcf250/82d7805a09d66e34dafc753cd2dfa63c6d9fe6c4c907d73a887f42fda4208e2d": none of the resources are being tracked.
Jun  8 17:26:20 node4 kubelet: E0608 17:26:20.975304    2241 remote_runtime.go:92] RunPodSandbox from runtime service failed: rpc error: code = Unknown desc = failed to start sandbox container for pod "hopping-meerkat-monocular-api-7477d54ff8-vwgk9": Error response from daemon: oci runtime error: container_linux.go:247: starting container process caused "process_linux.go:286: decoding sync type from init pipe caused \"read parent: connection reset by peer\""
Jun  8 17:26:20 node4 kubelet: E0608 17:26:20.975351    2241 kuberuntime_sandbox.go:54] CreatePodSandbox for pod "hopping-meerkat-monocular-api-7477d54ff8-vwgk9_default(21198eae-6a36-11e8-a0e1-525400bcf250)" failed: rpc error: code = Unknown desc = failed to start sandbox container for pod "hopping-meerkat-monocular-api-7477d54ff8-vwgk9": Error response from daemon: oci runtime error: container_linux.go:247: starting container process caused "process_linux.go:286: decoding sync type from init pipe caused \"read parent: connection reset by peer\""
Jun  8 17:26:20 node4 kubelet: E0608 17:26:20.975364    2241 kuberuntime_manager.go:647] createPodSandbox for pod "hopping-meerkat-monocular-api-7477d54ff8-vwgk9_default(21198eae-6a36-11e8-a0e1-525400bcf250)" failed: rpc error: code = Unknown desc = failed to start sandbox container for pod "hopping-meerkat-monocular-api-7477d54ff8-vwgk9": Error response from daemon: oci runtime error: container_linux.go:247: starting container process caused "process_linux.go:286: decoding sync type from init pipe caused \"read parent: connection reset by peer\""
Jun  8 17:26:20 node4 kubelet: E0608 17:26:20.975409    2241 pod_workers.go:186] Error syncing pod 21198eae-6a36-11e8-a0e1-525400bcf250 ("hopping-meerkat-monocular-api-7477d54ff8-vwgk9_default(21198eae-6a36-11e8-a0e1-525400bcf250)"), skipping: failed to "CreatePodSandbox" for "hopping-meerkat-monocular-api-7477d54ff8-vwgk9_default(21198eae-6a36-11e8-a0e1-525400bcf250)" with CreatePodSandboxError: "CreatePodSandbox for pod \"hopping-meerkat-monocular-api-7477d54ff8-vwgk9_default(21198eae-6a36-11e8-a0e1-525400bcf250)\" failed: rpc error: code = Unknown desc = failed to start sandbox container for pod \"hopping-meerkat-monocular-api-7477d54ff8-vwgk9\": Error response from daemon: oci runtime error: container_linux.go:247: starting container process caused \"process_linux.go:286: decoding sync type from init pipe caused \\\"read parent: connection reset by peer\\\"\""
Jun  8 17:26:21 node4 kubelet: W0608 17:26:21.008233    2241 pod_container_deletor.go:77] Container "82d7805a09d66e34dafc753cd2dfa63c6d9fe6c4c907d73a887f42fda4208e2d" not found in pod's containers
Jun  8 17:26:21 node4 dockerd: time="2018-06-08T17:26:21.317447560+08:00" level=error msg="Handler for POST /v1.26/containers/82d7805a09d66e34dafc753cd2dfa63c6d9fe6c4c907d73a887f42fda4208e2d/stop returned error: Container 82d7805a09d66e34dafc753cd2dfa63c6d9fe6c4c907d73a887f42fda4208e2d is already stopped"
Jun  8 17:26:21 node4 dockerd: time="2018-06-08T17:26:21.320776061+08:00" level=error msg="Handler for POST /v1.26/containers/e62b01e6cd396803ed6a0fa73d4d045a45e3d5b83c1f3b334941c046614cb317/stop returned error: Container e62b01e6cd396803ed6a0fa73d4d045a45e3d5b83c1f3b334941c046614cb317 is already stopped"
Jun  8 17:26:21 node4 dockerd: time="2018-06-08T17:26:21.325389638+08:00" level=error msg="Handler for POST /v1.26/containers/39e53f94d8fe4833639c71cb78890a2c900ce06784abb80bfe9b6d6fc2117e15/stop returned error: Container 39e53f94d8fe4833639c71cb78890a2c900ce06784abb80bfe9b6d6fc2117e15 is already stopped"
Jun  8 17:26:21 node4 dockerd: time="2018-06-08T17:26:21.328424932+08:00" level=error msg="Handler for POST /v1.26/containers/f84c20d67ab764705d662dfb571da77cf68d73d7c041dd070155e84081257452/stop returned error: Container f84c20d67ab764705d662dfb571da77cf68d73d7c041dd070155e84081257452 is already stopped"
Jun  8 17:26:21 node4 dockerd: time="2018-06-08T17:26:21.332781845+08:00" level=error msg="Handler for POST /v1.26/containers/d3f9f832146e7e37323896971a402e03135a806b3d9ae166563807b0ef2cd904/stop returned error: Container d3f9f832146e7e37323896971a402e03135a806b3d9ae166563807b0ef2cd904 is already stopped"
Jun  8 17:26:21 node4 dockerd: time="2018-06-08T17:26:21.335451694+08:00" level=error msg="Handler for POST /v1.26/containers/eba14f4f7b19b4d32836267203cf47112c732416c8ff508bfb5c04d47ebd42c3/stop returned error: Container eba14f4f7b19b4d32836267203cf47112c732416c8ff508bfb5c04d47ebd42c3 is already stopped"
Jun  8 17:26:21 node4 dockerd: time="2018-06-08T17:26:21.337755252+08:00" level=error msg="Handler for POST /v1.26/containers/28ce88a0ef40b7581b85ac6daf4fc922d4237846d0bae648c77ffa91dce53962/stop returned error: Container 28ce88a0ef40b7581b85ac6daf4fc922d4237846d0bae648c77ffa91dce53962 is already stopped"
Jun  8 17:26:21 node4 kernel: IPVS: Creating netns size=2040 id=13709
Jun  8 17:26:23 node4 kernel: SLUB: Unable to allocate memory on node -1 (gfp=0xd0)
Jun  8 17:26:23 node4 kernel:  cache: dentry(1683:698bb196b3516761d297e0eea39ab634cf228e24f7702a0b6cc7cd9f2964c068), object size: 192, buffer size: 192, default order: 1, min order: 0
Jun  8 17:26:23 node4 kernel:  node 0: slabs: 0, objs: 0, free: 0
Jun  8 17:26:24 node4 kernel: SLUB: Unable to allocate memory on node -1 (gfp=0xd0)
Jun  8 17:26:24 node4 kernel:  cache: sock_inode_cache(1683:698bb196b3516761d297e0eea39ab634cf228e24f7702a0b6cc7cd9f2964c068), object size: 632, buffer size: 640, default order: 3, min order: 0
Jun  8 17:26:24 node4 kernel:  node 0: slabs: 0, objs: 0, free: 0
Jun  8 17:26:24 node4 kernel: socket: no more sockets
Jun  8 17:26:24 node4 kernel: SLUB: Unable to allocate memory on node -1 (gfp=0xd0)
Jun  8 17:26:24 node4 kernel:  cache: sock_inode_cache(1683:698bb196b3516761d297e0eea39ab634cf228e24f7702a0b6cc7cd9f2964c068), object size: 632, buffer size: 640, default order: 3, min order: 0
```

### helm删除安装包，后台报容器找不到

```shell
Jun  8 09:34:01 node1 kubelet: W0608 09:34:01.504879    6698 iscsi_util.go:353] Warning: Unmount skipped because path does not exist: /var/lib/kubelet/plugins/kubernetes.io/iscsi/iface-/pvc-1fdcba47-6a36-11e8-a0e1-525400bcf250:3260-pvc-1fdcba47-6a36-11e8-a0e1-525400bcf250-lun-0
Jun  8 09:34:01 node1 kubelet: I0608 09:34:01.504935    6698 operation_generator.go:722] UnmountDevice succeeded for volume "pvc-1fdcba47-6a36-11e8-a0e1-525400bcf250" (UniqueName: "kubernetes.io/iscsi/pvc-1fdcba47-6a36-11e8-a0e1-525400bcf250:pvc-1fdcba47-6a36-11e8-a0e1-525400bcf250:0") on node "node1"
Jun  8 09:34:01 node1 kubelet: I0608 09:34:01.602502    6698 reconciler.go:297] Volume detached for volume "pvc-1fdcba47-6a36-11e8-a0e1-525400bcf250" (UniqueName: "kubernetes.io/iscsi/pvc-1fdcba47-6a36-11e8-a0e1-525400bcf250:pvc-1fdcba47-6a36-11e8-a0e1-525400bcf250:0") on node "node1" DevicePath ""
Jun  8 09:34:01 node1 kubelet: E0608 09:34:01.625770    6698 file.go:76] Unable to read manifest path "/etc/kubernetes/manifests": path does not exist, ignoring
Jun  8 09:34:02 node1 systemd-udevd: inotify_add_watch(7, /dev/dm-8, 10) failed: No such file or directory
Jun  8 09:34:03 node1 dockerd: time="2018-06-08T09:34:03.036612156+08:00" level=error msg="Handler for POST /v1.27/containers/03e6b2f25c5e22054aea7889ad4bad2dc447977d982ca41bf18835bd0a545e3d/stop returned error: Container 03e6b2f25c5e22054aea7889ad4bad2dc447977d982ca41bf18835bd0a545e3d is already stopped"
Jun  8 09:34:03 node1 kubelet: W0608 09:34:03.038790    6698 pod_container_deletor.go:77] Container "03e6b2f25c5e22054aea7889ad4bad2dc447977d982ca41bf18835bd0a545e3d" not found in pod's containers
Jun  8 09:34:03 node1 kubelet: W0608 09:34:03.751061    6698 prober.go:103] No ref for container "docker://505984b04068cf1c434bf698b1983c5221b17b0b3b607a6a6a74c9994b6cc37a" (hopping-meerkat-monocular-api-7477d54ff8-pr6df_default(21044e6b-6a36-11e8-a0e1-525400bcf250):monocular)
Jun  8 09:34:04 node1 systemd-udevd: inotify_add_watch(7, /dev/dm-8, 10) failed: No such file or directory
Jun  8 09:34:04 node1 kubelet: W0608 09:34:04.926450    6698 prober.go:103] No ref for container "docker://00efc8b41378ab54e052c216b8917d0f99b7be54454ea03c6f2d85a744971d01" (cautious-leopard-nginx-ingress-default-backend-7b6f5b895d-7wnb6_default(9f91a241-6a36-11e8-a0e1-525400bcf250):nginx-ingress-default-backend)
Jun  8 09:34:05 node1 systemd-udevd: inotify_add_watch(7, /dev/dm-8, 10) failed: No such file or directory
Jun  8 09:34:05 node1 dockerd: time="2018-06-08T09:34:05.180036294+08:00" level=error msg="Handler for POST /v1.27/containers/00646ff38996d12af3789a655560d941a210eba37f89801b9f62112c23921624/stop returned error: Container 00646ff38996d12af3789a655560d941a210eba37f89801b9f62112c23921624 is already stopped"
Jun  8 09:34:05 node1 kubelet: W0608 09:34:05.200959    6698 pod_container_deletor.go:77] Container "00646ff38996d12af3789a655560d941a210eba37f89801b9f62112c23921624" not found in pod's containers
Jun  8 09:34:05 node1 kubelet: W0608 09:34:05.201109    6698 pod_container_deletor.go:77] Container "a2aea037f2e0e9747dfa673c80aa492c2fefe1f0b576ce49985b3d1d0297234e" not found in pod's containers
Jun  8 09:34:05 node1 dockerd: time="2018-06-08T09:34:05.201948734+08:00" level=error msg="Handler for POST /v1.27/containers/a2aea037f2e0e9747dfa673c80aa492c2fefe1f0b576ce49985b3d1d0297234e/stop returned error: Container a2aea037f2e0e9747dfa673c80aa492c2fefe1f0b576ce49985b3d1d0297234e is already stopped"
Jun  8 09:34:05 node1 kubelet: W0608 09:34:05.220051    6698 pod_container_deletor.go:77] Container "36d58af5215d3d5b37d81a158008134016038b7623a3273d939fc68e3f50797c" not found in pod's containers
Jun  8 09:34:05 node1 kubelet: I0608 09:34:05.350221    6698 reconciler.go:191] operationExecutor.UnmountVolume started for volume "default-token-w5k7h" (UniqueName: "kubernetes.io/secret/9f91a241-6a36-11e8-a0e1-525400bcf250-default-token-w5k7h") pod "9f91a241-6a36-11e8-a0e1-525400bcf250" (UID: "9f91a241-6a36-11e8-a0e1-525400bcf250")
Jun  8 09:34:05 node1 kubelet: I0608 09:34:05.360010    6698 operation_generator.go:643] UnmountVolume.TearDown succeeded for volume "kubernetes.io/secret/9f91a241-6a36-11e8-a0e1-525400bcf250-default-token-w5k7h" (OuterVolumeSpecName: "default-token-w5k7h") pod "9f91a241-6a36-11e8-a0e1-525400bcf250" (UID: "9f91a241-6a36-11e8-a0e1-525400bcf250"). InnerVolumeSpecName "default-token-w5k7h". PluginName "kubernetes.io/secret", VolumeGidValue ""
Jun  8 09:34:05 node1 kubelet: I0608 09:34:05.450953    6698 reconciler.go:297] Volume detached for volume "default-token-w5k7h" (UniqueName: "kubernetes.io/secret/9f91a241-6a36-11e8-a0e1-525400bcf250-default-token-w5k7h") on node "node1" DevicePath ""
Jun  8 09:34:06 node1 systemd-udevd: inotify_add_watch(7, /dev/dm-8, 10) failed: No such file or directory
Jun  8 09:34:07 node1 systemd-udevd: inotify_add_watch(7, /dev/dm-8, 10) failed: No such file or directory
Jun  8 09:34:07 node1 kubelet: W0608 09:34:07.343321    6698 pod_container_deletor.go:77] Container "c5b2b1ad123f882e254887b8e95dae41a688a0bc8dcccc0ab60f60588615d861" not found in pod's containers
Jun  8 09:34:08 node1 kubelet: W0608 09:34:08.364529    6698 pod_container_deletor.go:77] Container "b53e18a4be83ed50333d89d239d8cc3472c078b3680ca036a1405d06dcbd510a" not found in pod's containers
Jun  8 09:34:13 node1 kubelet: W0608 09:34:13.751089    6698 prober.go:103] No ref for container "docker://505984b04068cf1c434bf698b1983c5221b17b0b3b607a6a6a74c9994b6cc37a" (hopping-meerkat-monocular-api-7477d54ff8-pr6df_default(21044e6b-6a36-11e8-a0e1-525400bcf250):monocular)
Jun  8 09:34:21 node1 kubelet: E0608 09:34:21.626130    6698 file.go:76] Unable to read manifest path "/etc/kubernetes/manifests": path does not exist, ignoring
Jun  8 09:34:23 node1 kubelet: W0608 09:34:23.750896    6698 prober.go:103] No ref for container "docker://505984b04068cf1c434bf698b1983c5221b17b0b3b607a6a6a74c9994b6cc37a" (hopping-meerkat-monocular-api-7477d54ff8-pr6df_default(21044e6b-6a36-11e8-a0e1-525400bcf250):monocular)
Jun  8 09:34:33 node1 kubelet: W0608 09:34:33.751145    6698 prober.go:103] No ref for container "docker://505984b04068cf1c434bf698b1983c5221b17b0b3b607a6a6a74c9994b6cc37a" (hopping-meerkat-monocular-api-7477d54ff8-pr6df_default(21044e6b-6a36-11e8-a0e1-525400bcf250):monocular)
Jun  8 09:34:35 node1 dockerd: time="2018-06-08T09:34:35.222209122+08:00" level=info msg="Container 505984b04068cf1c434bf698b1983c5221b17b0b3b607a6a6a74c9994b6cc37a failed to exit within 30 seconds of signal 15 - using the force"
Jun  8 09:34:41 node1 kubelet: E0608 09:34:41.626443    6698 file.go:76] Unable to read manifest path "/etc/kubernetes/manifests": path does not exist, ignoring
Jun  8 09:34:43 node1 kubelet: W0608 09:34:43.751096    6698 prober.go:103] No ref for container "docker://505984b04068cf1c434bf698b1983c5221b17b0b3b607a6a6a74c9994b6cc37a" (hopping-meerkat-monocular-api-7477d54ff8-pr6df_default(21044e6b-6a36-11e8-a0e1-525400bcf250):monocular)
Jun  8 09:34:45 node1 dockerd: time="2018-06-08T09:34:45.223348930+08:00" level=info msg="Container 505984b04068 failed to exit within 10 seconds of kill - trying direct SIGKILL"
```

通过`kubectl`查看容器被重启过很多次

```shell
[root@master2 ~]# kubectl get pods -o wide -w
NAME                                                             READY     STATUS        RESTARTS   AGE       IP             NODE
hopping-meerkat-monocular-api-7477d54ff8-pr6df                   0/1       Terminating   13         16h       10.244.10.97   node1
```

待确认找不到容器的原因，参考[kubernetes容器启动详解](https://blog.csdn.net/u010278923/article/details/72993499)。

### openebs存储卷不能被重启后的容器重新挂载

```shell
  Type     Reason                 Age   From                     Message
  ----     ------                 ----  ----                     -------
  Normal   Scheduled              2m    default-scheduler        Successfully assigned mongodb-76bd56459f-7plsl to node5
  Warning  FailedAttachVolume     2m    attachdetach-controller  Multi-Attach error for volume "pvc-2cd7dc93-6a38-11e8-a0e1-525400bcf250" Volume is already exclusively attached to one node and can't be attached to another
  Normal   SuccessfulMountVolume  2m    kubelet, node5           MountVolume.SetUp succeeded for volume "default-token-cv5rn"
  Warning  FailedMount            15s   kubelet, node5           Unable to mount volumes for pod "mongodb-76bd56459f-7plsl_kubeapps(7a944111-6abd-11e8-a0e1-525400bcf250)": timeout expired waiting for volumes to attach/mount for pod "kubeapps"/"mongodb-76bd56459f-7plsl". list of unattached/unmounted volumes=[data]
```

现象是节点的主机由于内核恐慌导致主机网卡奔溃，后丢失`kubernetes`容器，而存储卷是保存在该主机上，当服务容器再次启动一个后，重新挂载到该存储卷中，由于信息丢失，会导致失败。
