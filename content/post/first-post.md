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
![img.png](/img/img.png)
### k8s网络

## 2. 如何连接到k8s网络
### 时间