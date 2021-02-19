---
date: 2020-09-11
title: "虚拟机环境搭建-Ubuntu篇"
tags: ["Linux", "环境搭建","Ubuntu"]
---

## 虚拟机安装（略）

## Linux环境安装（略，使用Ubuntu系统）

## 设置主机名与IP的映射

### 查看当前操作系统版本

```bash
dgd@ubuntu:~$ uname -a
Linux ubuntu 5.3.0-53-generic #47~18.04.1-Ubuntu SMP Thu May 7 13:10:50 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
```

其中：

```bash
# 内核版本
dgd@ubuntu:~$ uname -v
#47~18.04.1-Ubuntu SMP Thu May 7 13:10:50 UTC 2020
# 内核发行号
dgd@ubuntu:~$ uname -r
5.3.0-53-generic
```

### 设置静态IP地址

> 为了方便在本地环境配置集群，故为虚拟机设置静态IP。

- 获取网卡名称

```bash
dgd@ubuntu:~$ ifconfig
ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.58.128  netmask 255.255.255.0  broadcast 192.168.58.255
        inet6 fe80::5440:7cd6:8318:ba45  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:7b:51:85  txqueuelen 1000  (以太网)
        RX packets 158592  bytes 223062647 (223.0 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 48831  bytes 3038718 (3.0 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (本地环回)
        RX packets 354  bytes 31719 (31.7 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 354  bytes 31719 (31.7 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

`ens33`为当前机器的网卡。

- 修改网卡配置文件

```bash
dgd@ubuntu:~$ sudo vi /etc/network/interfaces
# 输入以下内容:
# 对应网卡名称
auto ens33
# 设置为静态地址
iface ens33 inet static static
# 静态IP地址
address 192.168.58.151
# 子网掩码
netmask 255.255.255.0
# 网关
gateway 192.168.0.1
```

- 修改DNS配置

```bash
dgd@bigdata-01:~$ sudo vi /etc/resolv.conf 
# 输入以下内容
nameserver 8.8.8.8
```

> `ping www.baidu.com` 提示 `ping: unknown host www.baidu.com` :
>
> 如果`ping 8.8.8.8` 能通,可知是dns服务器没配好,
> 执行`sudo vi /etc/resolv.conf`,添加` nameserver 8.8.8.8` 即可。
>
> 如果`ping 8.8.8.8`也无法ping通知，检查虚拟机的网关配置地址是否跟Linux配置一样。

- 重启网络服务

  ```bash
  dgd@ubuntu:~$ sudo /etc/init.d/networking restart
  [ ok ] Restarting networking (via systemctl): networking.service.
  ```

  > 如果重启服务失败，尝试运行`sudo ifup -v ens33`看下错误提示，确认配置是否有错误。

- 重启后执行`ifconfig`查看IP地址是否已改变

### 用 Xshell 访问虚拟机

```bash
# 确认是否安装了SSH服务,如下提示表示没安装
dgd@bigdata-01:~$ ssh uname@localhost
ssh: connect to host localhost port 22: Connection refused
# 执行以下命令安装SSH
sudo apt-get install openssh-server
#安装完毕后ssh默认已启动。可以使用下述命令查看是否有进程在22端口上监听，即是否已启动：
dgd@bigdata-01:~$ netstat -nat | grep 22
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN 
```

### 修改主机名

```bas
# 查看主机名
dgd@ubuntu:~$ hostname
ubuntu
# 修改主机名
sudo vi /etc/hostname 
# 输入
bigdata-01.com
```

> 注意：并非所有的Linux发行主机名都存在`/etc/hostname`中，如Fedora发行版将主机名存放在`/etc/sysconfig/network`文件中。

### 设置主机名与IP的映射

```bash
sudo vi /etc/hosts
# 输入以下内容
bigdata-01.com 192.168.106.128
```

### 宿主机配置主机名与IP的映射 

这步完成后，就可以用`Xshell`通过主机名来访问指定机器了。

### 其它问题

#### 重启网络服务时,提示`this device is not active` 解决方法:

- 停止networkmaager服务

  ```bash
  sudo service NetworkManager stop
  ```

- 禁止自启动

  ```
  sudo chkconfig NetworkManager off
  ```

- 重启网络服务

  ```bash
  service network restart
  ```

原因是:安装了networmanager服务，同时两个服务network和networmanager管理网络导致冲突。