---
date: 2021-02-19
title: "虚拟机环境搭建-CentOS篇"
tags: ["Linux", "环境搭建","CentOS"]
---

##  VMware Workstation软件安装（略）

[VMware Workstation 15 Pro 密钥](https://www.jianshu.com/p/5dafcbab645d)。

## Linux环境安装（使用CentOS7）

- 安装好VMWare Workstation软件后，创建新的虚拟机：

  ![image-20210219095321162](/linux/创建虚拟机.png)

- 创建新的虚拟机：

  典型->安装客户机操作系统，选择CentOS 7 镜像文件->设置虚拟机存储位置->根据自己电脑硬盘大小指定磁盘容量->自定义硬件，根据电脑情况调整内存和处理器大小->完成，启动虚拟机。

  ![image-20210219100031133](/linux/创建虚拟机-1.png)

  ![image-20210219100226919](/linux/创建虚拟机-2.png)

  ![image-20210219100431642](/linux/创建虚拟机-3.png)

  ![image-20210219100701673](/linux/创建虚拟机-4.png)

  ![image-20210219100831590](/linux/创建虚拟机-5.png)

- 安装CentOS 7：

  选择语言->安装信息摘要，软件选择最小安装->开始安装->安装过程设置 root 用户密码（为方便学习记忆，可设置成6个0： 000000）->等待安装过程结束->重启CentOS 7。

  ![image-20210219102927508](/linux/安装CentOS7-1.png)

![image-20210219103227306](/linux/安装CentOS7-2.png)

![image-20210219103355276](/linux/安装CentOS7-3.png)

![image-20210219103921205](/linux/安装CentOS7-4.png)



## 环境配置

### 基本配置

- 操作重启之后，用 root 用户登录，输入密码：

  ![image-20210219104523389](/linux/基本配置-1.png)

- 编辑网卡配置文件：

  ```bash
  vi /etc/sysconfig/network-scripts/ifcfg-ens33
  ```

  ![image-20210219113059748](/linux/基本配置-2.png)

  将 `ONBOOT=no` 改成 `NOBOOT=yes`。

  ![image-20210219113416642](/linux/基本配置-3.png)

- 重启网络服务：

  ```bash
  service network restart
  ```

  ![image-20210219113617230](/linux/基本配置-4.png)

- 验证网络服务是否正常：

  ```bash
  ping www.baidu.com -c 3
  ```

  ![image-20210219113904797](/linux/基本配置-5.png)

- 安装 net-tools 工具：

  ```bash
  yum install -y net-tools
  ```

  ![image-20210219114201190](/linux/基本配置-6.png)

- 启动 `sshd` 服务（为了能使用远程连接工具）：

  ```bash
  service sshd start
  ```

  ![image-20210219114424355](/linux/基本配置-7.png)

- 安装文本编辑工具`vim`：

  ```bash
  yum install -y vim*
  ```

  ![image-20210219114643222](/linux/基本配置-8.png)

  - 也可以使用如下命令验证 `vim` 是否安装成功：

    ```bash
    rpm -qa | grep vim 
    ```

    ![image-20210219114831430](/linux/基本配置-9.png)

    

## 用 Xshell 访问虚拟机

- 查看当前虚拟机`IP`地址：

  ```bash
  ifconfig
  ```

  ![image-20210219115240226](/linux/用Xshell访问虚拟机-1.png)

- 下载并安装Xshell:

  [Xshell下载地址](https://www.netsarang.com/zh/xshell-plus-download/)。

- 打开`Xshell`并新建会话：

  ![image-20210219134314545](/linux/用Xshell访问虚拟机-2.png)

- 新建会话属性，主机一栏填入虚拟机 `IP` 地址：

  ![image-20210219134613248](/linux/用Xshell访问虚拟机-3.png)

- 用户身份验证，使用 `root` 用户登录，输入用户名和密码，最后连接：

  ![image-20210219134753064](/linux/用Xshell访问虚拟机-4.png)

- 连接虚拟机后，会弹出 `SSH安全警告`，选择接受并保存：

  ![image-20210219135022620](/linux/用Xshell访问虚拟机-5.png)

- 登录成功显示如下：

  ![image-20210219135204334](/linux/用Xshell访问虚拟机-6.png)

至此，虚拟机环境搭建-CentOS篇介绍完毕。



参考文档：

[CentOS7最小化安装后要做的事](https://www.cnblogs.com/trunkslisa/p/9493938.html)

[CentOS7 没有vim命令解决方法 -bash: vim: 未找到命令](https://blog.csdn.net/weixin_44057684/article/details/104967166)