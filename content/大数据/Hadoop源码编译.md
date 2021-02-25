---
date: 2021-02-23
title: "Hadoop源码编译"
tags: ["Hadoop", "源码编译"]
---

[toc]

## Linux环境安装

具体步骤参考[博客](https://blog.csdn.net/u012075383/article/details/113877300)。

## 前期准备工作

### 网络验证

执行以下命令，保证Linux能连接外网：

```bash
ping www.baidu.com -c 3
```

输出如下即确定网络连接没问题：

```bash
# 输出如下
PING www.a.shifen.com (14.215.177.39) 56(84) bytes of data.
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=1 ttl=128 time=9.72 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=2 ttl=128 time=9.47 ms
64 bytes from 14.215.177.39 (14.215.177.39): icmp_seq=3 ttl=128 time=9.57 ms

--- www.a.shifen.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 9.470/9.590/9.722/0.103 ms
```

### Jar 包准备

jdk-8u212-linux-x64.tar.gz

apache-maven-3.6.3-bin.tar.gz

### 虚拟机克隆



​    

