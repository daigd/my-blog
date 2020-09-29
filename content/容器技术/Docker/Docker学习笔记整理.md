---
title: "Docker学习笔记整理"
date: 2020-06-10
summary: "Docker学习笔记整理"
---



## docker入门知识

### docker概述

1. 什么是docker

   > [官网文档](https://docs.docker.com/get-started/)描述：`Docker is a platform for developers and sysadmins to build, run, and share applications with containers.`

   简单地说，docker是一个软件平台，它实现了软件应用的便捷构建、部署、运行和共享。

   

2. docker能干啥

   docker能干啥，不如说它解决什么了问题，想像一下，没有docker之前，如果我们要发布一个完成的java项目，比如是`jar`包或`war`包，是不是把打包好的文件交给运维，让其负责发布上线。正常情况姑且不谈，如果出了问题，譬如说，项目在开发或测试环境运行正常，结果到了正式环境报错，或者是，本地运行正常，换到其它环境就出Bug，类似的问题让人头大，而docker，就能杜绝上面类似问题的出现。

   

   上面提到的仅是docker的一个亮点，即**保证软件应用的运行环境一致**。除此之外，docker还能让开发/运维人员对**软件产品进行一次创建或配置，就能在任意地方正常运行**，再也不用担心配置不对的问题，有了这一特性的支持，应用的迁移、扩展不是分分钟的事？因此，利用docker，可以**极其方便地对应用进行迁移，扩展和维护**。

   

3. 简单说下docker的历史

4. docker和虚拟机有啥区别

5. 为啥要用docker?知道DevOps吗？

6. docker的基本组成

7. 说说docker的底层原理

### docker安装

#### docker如何安装

> 以下演示操作系统为Ubuntu,详情可参考[官网](https://docs.docker.com/engine/install/ubuntu/)。

移除旧版本

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
```

更新`apt`软件包索引

```bash
sudo apt-get update
```

添加几个小工具,使得`apt`支持Https获取软件包

```bash
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

添加Docker官方库的GPG密钥

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

将Docker仓库添加到软件源中

```bash
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

添加成功后更新软件包缓存

```bash
sudo apt-get update
```

安装Docker,默认安装最新版本

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

验证Docker是否安装成功

```bash
sudo docker run hello-world
```

> 当看到以下输出时表明Docker已安装成功。

```bash
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

指定用户加入Docker分组

> 为了避免每次使用Docker命令都加上sudo,可以创建Docker分组，并将当前用户加入。

```bash
# 查看是否已有docker分组
sudo cat /etc/group | grep docker
# 如果未创建docker分组,执行下面命令创建
sudo groupadd docker 
# 应用用户加入docker用户组
sudo usermod -aG docker ${USER}
# 重启docker服务
sudo systemctl restart docker
# 切换账号或退出当前账号再重新登录即可
# 执行以下命令验证当前用户加入分组是否成功
docker version
```

#### 讲讲docker运行run命令后发生了什么？

#### 运行docker为啥比虚拟机快

### docker常用命令

1. docker镜像的常用命令有哪些
2. docker容器的常用命令有哪些
3. 谈下docker的其它常用命令，如查看日志，元数据，进程等

### 理解docker镜像

1. 什么是docker镜像
2. 说说镜像的加载原理
3. 谈谈对镜像文件分层的理解
4. 怎么提交一个新镜像

## docker进阶

### docker数据卷

1. 什么是docker数据卷
2. 谈谈容器数据卷的理解
3. 如何使用数据卷
4. 什么是具名、匿名挂载
5. 如何指定数据卷的读写权限
6. 什么是数据卷容器？容器如何实现数据共享或备份？

### DockerFile

1. 什么是DockerFile
2. 说说DockerFile的构建过程
3. docker build命令最后的点有什么含义
4. 说说DockerFile的常用命令
5. 写一个tomcat镜像的DockerFile
6. 简单说下发布镜像的过程

### docker网络

1. docker如何处理网络访问的

2. 使用dockr run 命令时，若使用参数`--link`，背后发生了啥？

3. 如何创建自定义docker网络

   首先用```docker network --help``` 查看下该命令的用法 ：

   ```shell
   # 查看 docker network 命令的使用
   Usage:  docker network COMMAND
   
   Manage networks
   
   Commands:
    # 连接一个容器到指定网络
     connect     Connect a container to a network
     # 创建一个网络
     create      Create a network
     # 停掉一个网络
     disconnect  Disconnect a container from a network
     # 查看网络的详情
     inspect     Display detailed information on one or more networks
     ls          List networks
     prune       Remove all unused networks
     rm          Remove one or more networks
   
   Run 'docker network COMMAND --help' for more information on a command.
   ```

   可以看到，创建docker网络的命令是`docker network create`，再来看下`docker network create`的具体用法：

   ```shell
   # 创建一个docker网络
   Usage:  docker network create [OPTIONS] NETWORK
   
   Create a network
   
   Options:
         --attachable           Enable manual container attachment
         --aux-address map      Auxiliary IPv4 or IPv6 addresses used by Network driver (default map[])
         --config-from string   The network from which copying the configuration
         --config-only          Create a configuration only network
         # 指定网络的驱动模式，默认为桥接模式
     -d, --driver string        Driver to manage the Network (default "bridge")
         # 指定网关
         --gateway strings      IPv4 or IPv6 Gateway for the master subnet
         --ingress              Create swarm routing-mesh network
         --internal             Restrict external access to the network
         --ip-range strings     Allocate container ip from a sub-range
         --ipam-driver string   IP Address Management Driver (default "default")
         --ipam-opt map         Set IPAM driver specific options (default map[])
         --ipv6                 Enable IPv6 networking
         --label list           Set metadata on a network
     -o, --opt map              Set driver specific options (default map[])
         --scope string         Control the network's scope
         # 指定子网掩码，格式遵从CIDR
         --subnet strings       Subnet in CIDR format that represents a network segment
   ```

   可以看到，创建网络的参数比较多，但是只要指定`subnet`和`gateway`两个参数就可创建一个网络了，如下：

   ```shell
   # 创建一个名为mynet的网络，子网掩码指定IP地址的前缀位数为24位，即该网络最多只能分配 256个地址
   docker network create  --subnet 192.161.0.0/24 --gateway 192.161.0.1 mynet
   # 查看网络信息，可以看到该网络是按我们的要求创建出来的
   docker network inspect mynet
   
   [
       {
           "Name": "mynet",
           "Id": "39a5176f201e500e85521e2960a025e124c897f7da566ef2cd7d2bd97bc190f5",
           "Created": "2020-06-08T11:12:43.944734557+08:00",
           "Scope": "local",
           "Driver": "bridge",
           "EnableIPv6": false,
           "IPAM": {
               "Driver": "default",
               "Options": {},
               "Config": [
                   {
                       "Subnet": "192.161.0.0/24",
                       "Gateway": "192.161.0.1"
                   }
               ]
           },
           "Internal": false,
           "Attachable": false,
           "Ingress": false,
           "ConfigFrom": {
               "Network": ""
           },
           "ConfigOnly": false,
           "Containers": {},
           "Options": {},
           "Labels": {}
       }
   ]
   ```

   

4. 说说docker的几种网络模式

5. 自定义docker网络的好处是啥

6. 连接不同自定义网络的容器，如何实现网络互联







