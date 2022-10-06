---
title: "本地如何直连kubernetes环境"
date: 2022-09-25T22:50:43+08:00
tags: ["docker","kubernetes"]
author: "zibuyu28"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "连接k8s网络"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/zibuyu28.github.io/content"
    Text: "edit" # edit text
    appendFilePath: false # to append file path to Edit link
---

## 0. 由来
* 本地开发可以在本机搭建solo环境（各种依赖服务）
* 但是在本地资源比较紧张的时候，就会在开发主机上使用docker部署的微服务系统
  * docker部署方式会将各个微服务的服务端口开放置宿主机
  * 服务内注册的地址形式 {宿主机ip:映射端口}
  * 本地开发环境可以直连开发主机上的微服务
* 当测试在测试环境（kubernetes环境）测试，出现问题之后找bug本地无法直连测试环境，只能依靠日志（需要添加日志重新部署）
* 希望可以在本地直接连接kubernetes环境

## 1. 了解网络
### 容器网络
* 本地启动docker之后，会有一个docker0网卡，容器内会有一个eth0网卡，但是这两个网卡是不同的，有一个虚拟设备veth-pair，这就类似一根网线一端插在容器内的网卡，一端插在docker0网卡，这样就实现了容器和宿主机的互通
* 本地启动docker之后可以看到一个docker0网卡，及路由信息
```shell
➜  ~ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         gateway         0.0.0.0         UG    100    0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.19.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-efd21faa9c32
172.20.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-2696f6e7def3
172.21.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-d27e3bb7d16b
172.22.71.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
172.24.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-f0f835ebbad4

docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:13ff:fe29:e696  prefixlen 64  scopeid 0x20<link>
        ether 02:42:13:29:e6:96  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 67  bytes 7346 (7.1 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
vethdf3b559: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::3cd1:7dff:fe6e:5269  prefixlen 64  scopeid 0x20<link>
        ether 3e:d1:7d:6e:52:69  txqueuelen 0  (Ethernet)
        RX packets 5160794  bytes 848623285 (809.3 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 4654270  bytes 2590776548 (2.4 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
* 容器里面有一个eth0网卡，及路由信息
```shell
bash-5.0# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         172.21.0.1      0.0.0.0         UG    0      0        0 eth0
172.21.0.0      *               255.255.0.0     U     0      0        0 eth0

bash-5.0# ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:15:00:0C
          inet addr:172.21.0.12  Bcast:172.21.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:63050462 errors:0 dropped:0 overruns:0 frame:0
          TX packets:66632046 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:28284576109 (26.3 GiB)  TX bytes:6081185686 (5.6 GiB)
```
* 也就是容器内默认的ip请求都会通过eth0 通过 veth-pair到达宿主机的网卡，然后将请求传到docker0，最后发送到对的目的地
* 如果是两个容器这样就可以通过docker0进行通信
![img.png](/post1/img1.png)
* 如果想实现跨主机的容器通信，那么和两个容器通信类似，如果在两台宿主机上增加了类似docker0这样的网桥是不是就可以实现两个跨主机容器通信呢
![img.png](/post1/img2.png)
* 三种主流容器跨主机网络方案：
  * VXLAN（Virtual Extensible LAN）：虚拟可扩展局域网，这个是linux内核支持的一个网络虚拟化技术，所以VXLAN可以在内核态完成数据的封包和解封的过程
    * ![img.png](/post1/img4.png)
    * 只有容器内是用户态，而docker0，flannel.1，eth0都是内核态，所以这样提高了效率
    * VXLAN 的覆盖网络的设计思想是：在现有的三层网络之上，“覆盖”一层虚拟的、由内核 VXLAN 模块负责维护的二层网络，使得连接在这个 VXLAN 二层网络上的“主机”（虚拟机或者容器都可以）之间，可以像在同一个局域网（LAN）里那样自由通信
    * 图中的flannel.1设备就是这个虚拟隧道的两端的设备
  * host-gw
    * 工作原理：将每个 Flannel 子网（Flannel Subnet，比如：10.244.1.0/24）的“下一跳”，设置成了**该子网对应**的宿主机的 IP 地址
    * 此模式能够正常工作的核心，就在于 IP 包在封装成帧发送出去的时候，会使用路由表里的“下一跳”来设置目的 MAC 地址。这样，它就会经过二层网络到达目的宿主机。
    * 所以此模式必须要求集群宿主机之间是二层连通的
    * 但是二层不同的情况是广泛存在的
  * UDP：性能最差已经弃用，因为数据从发出到目的地总共会经历三次用户态和内核态之间的数据拷贝
    * ![img.png](/post1/img3.png)
  * 用户的容器都连接在 docker0 网桥上。而网络插件则在宿主机上创建了一个特殊的设备（UDP 模式创建的是 TUN 设备，VXLAN 模式创建的则是 VTEP 设备），docker0 与这个设备之间，通过 IP 转发（路由表）进行协作。
### k8s网络
* k8s不使用docker0网卡，通过CNI接口来维护了 CNI 网桥，默认名称 cni0
* k8s网络主要使用领域内"龙头老大" Calico
* Calico的网络解决方案和host-gw几乎完全一样
  * ![img.png](/post1/img6.png)
* 添加很多如下格式的路由
```shell
    < 目的容器 IP 地址段 > via < 网关的 IP 地址(目的容器所在宿主机的 IP 地址) > dev eth0
```
* 不同于Flannel通过etcd维护路由信息，calico会自动地在整个集群中分发路由信息（BGP边界网关协议）
  * BGP：每个边界网关上都会运行着一个小程序，它们会将各自的路由表信息，通过 TCP 传输给其他的边界网关。而其他边界网关上的这个小程序，则会对收到的这些数据进行分析，然后将需要的信息添加到自己的路由表里
  * 所谓 BGP，就是在大规模网络中实现节点路由信息共享的一种协议
* 所以Calico就会通过CNI插件和k8s的CNI网桥对接
* 当k8s两台宿主机之间不再一个子网内（二层网络不通），这个时候就需要让Calico开启IPIP模式
  * ![img.png](/post1/img5.png)
### k8s 路由解析
* 可以知道k8s网络内有一个内部dns服务器，kube-dns地址
  * service解析到service的虚拟ip（vip），然后通过iptables内信息指向另外几个pod的记录，最终通过选择pod的记录指向pod的ip
* service 的集群ip是VIP
## 2. 如何连接到k8s网络
### 几种方案及实践
* 通过加入calico网络，直接打通本地主机和k8s集群
  * 思考：
      * 通过calico发出的请求我可以通过tun0设备直接转发到集群宿主机，进入集群主机之后我的所有ip都是可以访问通的
      * service 如何解析出pod ip：只要连接到 k8s dns服务器作为我本地的自定义dns就可以解析出对应ip
      * k8s 的 dns 是通过容器部署的，但是只要我能打通与集群的宿主机网络，是否可以直接使用这个dns
      * 这样整个逻辑就是通的
  * 确认k8s的calico网络开启了ipip模式，因为本地机器和集群宿主机完全不在同一个网段
  * 在k8s内增加一个虚拟节点，主要用于calico-node注册节点信息
    ```shell
    kubectl apply -f - <<EOF
    kind: Node
    apiVersion: v1
    metadata:
      name: node-ext
    EOF
    ```
  * 宿主机本地启动calico-node容器，让本地也加入calico网络，指定kubeconfig文件即可，需要写本地ip
    ```shell
    docker run -d --net=host --privileged \
     --name=calico-node \
     -e NODENAME="node-ext" \
     -e IP="172.22.71.24x" \
     -e CALICO_NETWORKING_BACKEND=bird \
     -e NO_DEFAULT_POOLS=true \
     -e DATASTORE_TYPE=kubernetes \
     -e KUBECONFIG=/var/config/config \
     -e CALICO_STARTUP_LOGLEVEL=DEBUG \
     -v /etc/calico/calico-kubeconfig:/calico/calico-kubeconfig \
     -v /var/log/calico:/var/log/calico \
     -v /var/lib/calico:/var/lib/calico \
     -v /var/run/calico:/var/run/calico \
     -v /run/docker/plugins:/run/docker/plugins \
     -v /lib/modules:/lib/modules \
     -v /home/blocface/.kube/:/var/config/ \
     docker.io/calico/node:release-v3.18
    ```
  * 实践之后发现mac本地无法同步到对应的路由信息
    * 猜测：因为mac的docker-desktop是启动了一个虚拟机来启动docker的，所以可能信息被同步到那个小虚拟机内部了
  * 但是在linux机器上搭建calico-node之后，路由信息正确的出现在路由信息中
  * 使用`ping 容器ip`可以ping通
  * 修改 /etc/resolv.cnf 增加dns服务器
  * 尝试 `ping service-name` 失败
    ```shell
    ~ ping blocface-core.blocface-test.svc.cluster.local
    PING blocface-core.blocface-test.svc.cluster.local (192.168.154.217) 56(84) bytes of data.
    ```
  * 可以看到 service 对应的虚拟ip（192.168.154.17）已经解析出来了，但是新增的节点和集群内节点 192.168 这个网段没有通，导致无法访问到这个service的ip，最终无法正确访问到pod
  * 按道理如果这个 service 的 ip 网段不再192.168 应该是可以实现访问成功的
  * 从结果可以看出这个方式的局限性比较大，还会在k8s集群内增加一个不可用的节点（仅仅为了网络通信），calico是不推荐这么做的
  * 参考[https://lqingcloud.cn/post/calico-03/](https://lqingcloud.cn/post/calico-03/)
* 使用telepresence来打通本地和k8s的网络
  * 下载`telepresence`工具
  ```shell
  amd64 : sudo curl -fL https://app.getambassador.io/download/tel2/darwin/amd64/latest/telepresence -o /usr/local/bin/telepresence && sudo chmod a+x /usr/local/bin/telepresence
  arm64 : sudo curl -fL https://app.getambassador.io/download/tel2/darwin/arm64/latest/telepresence -o /usr/local/bin/telepresence && sudo chmod a+x /usr/local/bin/telepresence
  ```
  * 安装`triffic manager`
  ```shell
  telepresence helm install --kubeconfig=~/./kube/config # kubeconfig 默认使用 ~/.kube/config 如有特殊可以指定
  ```
  * 启动`telepresence`
  ```shell
  telepresence connect --kubeconfig=~/.kube/config # kubeconfig 默认使用 ~/.kube/config 如有特殊可以指定
  ```
  * 连接成功之后，就可以直接访问k8s网络了
  ```shell
  ping xxxx-service.<namespace>
  ```