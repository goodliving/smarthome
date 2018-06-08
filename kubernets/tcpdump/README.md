### 概述

这是关于在`kubernetes`的各种网络模型的抓包记录与分析。

### kube-router

首先部署好一个nginx的`pod`，之后在节点使用`tcpdump`进行抓包

```shell
[root@master2 demo]# curl 10.244.3.10

[root@node3 centos]# tcpdump -i any port 80
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
11:48:15.963401 IP 192.168.200.21.56888 > 10.244.3.10.http: Flags [S], seq 779547779, win 14600, options [mss 1460,sackOK,TS val 3019404299 ecr 0,nop,wscale 7], length 0
11:48:15.963613 IP 192.168.200.21.56888 > 10.244.3.10.http: Flags [S], seq 779547779, win 14600, options [mss 1460,sackOK,TS val 3019404299 ecr 0,nop,wscale 7], length 0
11:48:15.963616 IP 192.168.200.21.56888 > 10.244.3.10.http: Flags [S], seq 779547779, win 14600, options [mss 1460,sackOK,TS val 3019404299 ecr 0,nop,wscale 7], length 0
11:48:15.963777 IP 10.244.3.10.http > 192.168.200.21.56888: Flags [S.], seq 1256154880, ack 779547780, win 28960, options [mss 1460,sackOK,TS val 3611468166 ecr 3019404299,nop,wscale 7], length 0
11:48:15.963786 IP 10.244.3.10.http > 192.168.200.21.56888: Flags [S.], seq 1256154880, ack 779547780, win 28960, options [mss 1460,sackOK,TS val 3611468166 ecr 3019404299,nop,wscale 7], length 0
11:48:15.963799 IP 10.244.3.10.http > 192.168.200.21.56888: Flags [S.], seq 1256154880, ack 779547780, win 28960, options [mss 1460,sackOK,TS val 3611468166 ecr 3019404299,nop,wscale 7], length 0
11:48:15.968201 IP 192.168.200.21.56888 > 10.244.3.10.http: Flags [.], ack 1, win 115, options [nop,nop,TS val 3019404303 ecr 3611468166], length 0
11:48:15.968251 IP 192.168.200.21.56888 > 10.244.3.10.http: Flags [.], ack 1, win 115, options [nop,nop,TS val 3019404303 ecr 3611468166], length 0
11:48:15.968257 IP 192.168.200.21.56888 > 10.244.3.10.http: Flags [.], ack 1, win 115, options [nop,nop,TS val 3019404303 ecr 3611468166], length 0
11:48:15.968293 IP 192.168.200.21.56888 > 10.244.3.10.http: Flags [P.], seq 1:76, ack 1, win 115, options [nop,nop,TS val 3019404303 ecr 3611468166], length 75: HTTP: GET / HTTP/1.1
11:48:15.968316 IP 192.168.200.21.56888 > 10.244.3.10.http: Flags [P.], seq 1:76, ack 1, win 115, options [nop,nop,TS val 3019404303 ecr 3611468166], length 75: HTTP: GET / HTTP/1.1
11:48:15.968320 IP 192.168.200.21.56888 > 10.244.3.10.http: Flags [P.], seq 1:76, ack 1, win 115, options [nop,nop,TS val 3019404303 ecr 3611468166], length 75: HTTP: GET / HTTP/1.1
11:48:15.968558 IP 10.244.3.10.http > 192.168.200.21.56888: Flags [.], ack 76, win 227, options [nop,nop,TS val 3611468170 ecr 3019404303], length 0
11:48:15.968571 IP 10.244.3.10.http > 192.168.200.21.56888: Flags [.], ack 76, win 227, options [nop,nop,TS val 3611468170 ecr 3019404303], length 0
11:48:15.968586 IP 10.244.3.10.http > 192.168.200.21.56888: Flags [.], ack 76, win 227, options [nop,nop,TS val 3611468170 ecr 3019404303], length 0
```

