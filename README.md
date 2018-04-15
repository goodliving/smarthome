###  概述

这是关于智能家居的项目，主要实现厨房、客厅、卧室、阳台等家装智能系统，以及家庭种植计划。使用树莓派系统作为中控系统，通过容器docker来加载本地项目，物理设备设计成模块化等。

### 技术栈

* Python
* Go
* Raspberry
* Docker
* Linux
* Electronic
* WeChat(wxpy)
* gobot
* ceph
* kubernetes
* Istio
* hubot
* prometheus

### 功能

从家庭区域来说，主要分为以下几个部分

- [ ] 阳台
- [ ] 客厅
- [ ] 卧室
- [ ] 厨房
- [ ] 浴室


#### 阳台

在这里是整个种植区，物理设备来说，是一个六层架设，每层是100*30cm的种植块，其中按照光照强度分为不同蔬菜种类，养分以及水源通过特定管道进行输送，同时监控生长环境，统计各个环境变量的关系，制作生长周期配方以及模块化，方便各个环境的转移，同时个人可以上传特定的配方供别人参考以及下载。

种植模型从功能上划分有承载层、养料层、监控层、web通信层、客户端层，其中承载层是指物理模具设计以及土壤规划，养料层是指水分、阳光、温度、空气，控制层是指接收到web层传递过来的生长配方控制养料层的各个变量值以及实时记录当前变量值，web通信层是指收集到控制层的数据进行保存以及人工算法判断蔬菜生产情况，客户端层展示数据，同时也可以进行人工微调。

支持小规模种植或者集中种植。

### 客厅

这里面是家庭影院的组成部分，包括投影仪、智能电视、音响、中央空调等。

### 卧室

主要是用来控制灯光、温度、影视等。

### 厨房

主要是用来控制灯光、温度、水检测、燃气、抽烟机等。

### 浴室

主要是用来控制灯管、温度、热水加热器、水冲洗等。

### 室外

主要是指家庭汽车信息交互功能，导航、音乐、电话等。

### 设计原理图

整个系统中有一个核心部件用来与各个子系统进行交互，数据采集与分析，做出结果判断后，通过消息部件发送消息推送以及终端显示，整个系统部署采用微服务结果，高可用设计。以下是整个系统部件的大概规划

* 核心部件corecenter
* 消息部件message
* 客厅部件darwroom
* 卧室部件bedroom
* 厨房部件cookroom
* 浴室部件bathroom
* 阳台部件plantroom
* 室外部件outroom

消息推送包括家庭信息以及家庭外信息，包括天气、交通推送等。该功能通过智能运维机器人来监听，实现基础设施的监控以及事件的回应。

整个系统有个中间层，获取到客户端请求，与后台组件交互获取到数据，然后将数据返回给前端渲染。

整个系统有个数据展示层，使用prometheus做监控层。