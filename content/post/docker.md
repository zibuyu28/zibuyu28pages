---
title: "Docker 基础知识"
date: 2022-10-09T12:21:41+08:00
tags: ["docker"]
author: "zibuyu28"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "docker 基础知识"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
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
    URL: "https://github.com/zibuyu28/zibuyu28.github.io/content"
    Text: "edit" # edit text
    appendFilePath: true # to append file path to Edit link
---
> docker 容器的本质是进程
* 一张图理解容器和虚拟机的区别
![img.png](/docker/img1.png)
![img.png](/docker/img.png)
* 对比这两张图可以发现第二张图更为准确，因为docker engine只是一个服务工具而已，所有的容器进程还是运行在宿主机上的
### docker 如何实现隔离
* 我们知道docker容器内部看到的pid和宿主机的pid是相互隔离的，这个是通过 Linux 的 Namespace 技术实现的
* 当系统调用 clone 方法创建进程的时候，增加了一个 CLONE_NEWPID 参数，这样创建的进程就会"看到"一个全新的空间，这样当前进程的 pid 就是 1，但是在宿主机上这个进程的pid是真实的pid（比如100）
* 上述的就是 PID Namespace，类似的还有Mount，UTS，IPC，Network，User这些Namespace，都是类似"障眼法"的功能

### docker 如何实现限制
* 因为所有的容器进程都是运行在宿主机上的，所以各个容器都是共用宿主机操作系统内核的，包括cpu，内存等资源
* 使用Linux的Cgroup功能来限制进程的资源使用。Linux Cgroups 的全称是 Linux Control Group。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等
* 容器是一个"单进程"模型

### docker 如何实现文件读写
* 容器内目录文件需要通过Mount Namespace来进行隔离， Mount Namespace 修改的，是容器进程对文件系统"挂载点"的认知
* 通过Mount Namespace 可以改变进行对 "/" 目录的认知，即改变进程的根文件系统 （change root file system）
* 而这个挂载在容器根目录上、用来为容器进程提供隔离后执行环境的文件系统，就是所谓的"容器镜像"。它还有一个更为专业的名字，叫作：rootfs（根文件系统）。
* rootfs 只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。所以说，rootfs 只包括了操作系统的"躯壳"，并没有包括操作系统的"灵魂"

### docker 常见容器的基本步骤
* 启动Linux Namespace 配置
* 设置指定的 Cgroups 参数
* 切换进程的根目录（Change root）

### docker 镜像来解决每次启动容器都需要拷贝 rootfs 文件的问题
* docker的镜像设计引入了层（layer）的概念，也就是用户制作镜像的每一步骤都是一层，都是增量的rootfs
* 那每一层都会有不通的东西和相同的东西，这些东西怎么从层（垂直结构）的概念到最后的 目录结构（水平结构）的呢
  * 联合文件系统（UnionFS）来解决这个问题，比如又两个文件A，B
    ```shell
    $ tree
    .
    ├── A
    │  ├── a
    │  └── x
    └── B
      ├── b
      └── x
    ```
  * 使用联合挂载的方式挂载到公共目录C
    ```shell
    $ mkdir C
    $ mount -t aufs -o dirs=./A:./B none ./C
    # 再次查看C目录，发现 a，b文件在同一层级，x文件被合并了
    $ tree ./C
    ./C
    ├── a
    ├── b
    └── x
    ```
* 所以当一个容器启动的时候就会在容器专用文件夹下进行将镜像的多层以联合文件的方式挂载到这个容器专用文件夹下，就是容器看到的所有的文件内容
  * 查看挂载的层信息
  ```shell
   $ cat /sys/fs/aufs/si_972c6d361e6b32ba/br[0-9]*
   /var/lib/docker/aufs/diff/6e3be5d2ecccae7cc...=rw
   /var/lib/docker/aufs/diff/6e3be5d2ecccae7cc...-init=ro+wh
   /var/lib/docker/aufs/diff/32e8e20064858c0f2...=ro+wh
   /var/lib/docker/aufs/diff/2b8858809bce62e62...=ro+wh
   /var/lib/docker/aufs/diff/20707dce8efc0d267...=ro+wh
   /var/lib/docker/aufs/diff/72b0744e06247c7d0...=ro+wh
   /var/lib/docker/aufs/diff/a524a729adadedb90...=ro+wh
  ```
  * 总体来看这个容器的rootfs的组成由三部分组成
  ![img_1.png](/docker/img_1.png)
    1. 只读层
       * 挂载方式都是只读的（ro+wh，即 readonly+whiteout） 
    2. 可读写层
       * 这一层在文件没有写入之前是空的，一旦由新增就会以增量的方式加入到这个层里面
       * 那如果是删除只读层里面一个文件呢，那就会创建一个whiteout文件，吧只读层里面的文件"遮挡"起来
       * 比如，你要删除只读层里一个名叫 foo 的文件，那么这个删除操作实际上是在可读写层创建了一个名叫.wh.foo 的文件。这样，当这两个层被联合挂载之后，foo 文件就会被.wh.foo 文件“遮挡”起来，“消失”了。这个功能，就是“ro+wh”的挂载方式，即只读 +whiteout 的含义。我喜欢把 whiteout 形象地翻译为：“白障”。
       * 那么如果你是要修改只读层的文件呢，这时候就需要在读写层先拷贝，然后修改，最后挂载的时候会因为层的顺序，吧原有的只读层文件给"遮盖"掉，这样就达到了修改文件的目的
    3. Init层
       * Init 层是 Docker 项目单独生成的一个内部层，专门用来存放 /etc/hosts、/etc/resolv.conf 等信息。