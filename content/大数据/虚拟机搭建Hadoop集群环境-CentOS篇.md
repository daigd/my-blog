---
date: 2021-02-20
title: "虚拟机搭建Hadoop集群环境-CentOS篇"
tags: ["Linux", "环境搭建","CentOS"]
---

## Linux环境安装

具体步骤参考[博客](https://blog.csdn.net/u012075383/article/details/113877300)。

## 虚拟机环境准备

### 虚拟机克隆

在前一步骤中准备好安装了CentOS 7 的虚拟机，克隆一个虚拟机出来，过程如下图：

![image-20210220102925391](/hadoop/克隆虚拟机-1.png)

![image-20210220103031610](/hadoop/克隆虚拟机-2.png)

![image-20210220103118652](/hadoop/克隆虚拟机-3.png)

![image-20210220103152242](/hadoop/克隆虚拟机-4.png)

![image-20210220103355478](/hadoop/克隆虚拟机-5.png)

![image-20210220103547163](/hadoop/克隆虚拟机-6.png)

### 修改虚拟机IP

- 克隆完成后，启动`bigdata-101`虚拟机，使用`root`用户登录，修改虚拟机的`IP`地址

  ![image-20210220104011779](/hadoop/修改虚拟机IP-1.png)

  - 编辑`/etc/sysconfig/network-scripts/ifcfg-ens33`文件，修改成如下内容：

    ```bash
    vim /etc/sysconfig/network-scripts/ifcfg-ens33
    # 修改内容如下
    TYPE=Ethernet
    BOOTPROTO=static # 设置静态IP
    NAME=ens33
    DEVICE=ens33
    ONBOOT=yes
    IPADDR=192.168.1.101 # 自定义IP地址
    PREFIX=24
    GATEWAY=192.168.1.2
    DNS1=192.168.1.2
    ```

    

    ![image-20210220104439994](/hadoop/修改虚拟机-2.png)

    ![image-20210220105233994](/hadoop/修改虚拟机-3.png)

  - 设置虚拟机虚拟网络编辑器，编辑->虚拟网络编辑器->VMnet8->更改配置：

    ![image-20210220105352550](/hadoop/修改虚拟机IP-4.png)

    ![image-20210220105928304](/hadoop/修改虚拟机IP-5.png)

    将子网IP按下图调整，最后点击 NAT 设置：

    

    ![image-20210220141112465](/hadoop/修改虚拟机IP-6.png)

    ![image-20210220110426799](/hadoop/修改虚拟机IP-7.png)

  - 设置VMnet8 属性，保证默认网关、首先DNS服务器和Linux环境配置一致

    ![image-20210220110550653](/hadoop/修改虚拟机IP-8.png)

    ![image-20210220110724274](/hadoop/修改虚拟机IP-9.png)

    ![image-20210220110813060](/hadoop/修改虚拟机IP-10.png)

    ![image-20210220110925504](/hadoop/修改虚拟机IP-11.png)

    ![image-20210220111036851](/hadoop/修改虚拟机IP-12.png)

### 修改主机名

- 查看当前主机名

  ```bash
  hostname
  ```

  ![image-20210220111608753](/hadoop/修改主机名-1.png)

- 执行`vim /etc/sysconfig/network`,输入以下内容：

  ```bash
  HOSTNAME=bigdata101
  ```

  ![image-20210220111858883](/hadoop/修改主机名-2.png)

- 执行 `vim /etc/hosts`，加入以下内容：

  ```bash
  192.168.1.101 bigdata101
  192.168.1.102 bigdata102
  192.168.1.103 bigdata103
  ```

  ![image-20210220112057192](/hadoop/修改主机名-3.png)

  ![image-20210220112229009](/hadoop/修改主机名-4.png)

  ![image-20210220112314496](/hadoop/修改主机名-5.png)

### 创建普通用户

- 创建一个普通用户用于日常操作，用户名为`bigdata`，为方便学习记忆，密码也可设置为`bigdata`：

  ```bash
  # 添加用户
  useradd bigdata
  # 设置密码
  passwd bigdata
  ```

  ![image-20210220112743743](/hadoop/创建普通用户-1.png)

- 给新添加的 `hadoop`用户配置 root 权限：

  ```bash
  # 添加写入模式
  chmod u+w /etc/sudoers
  # 在 root ALL=(ALL)	ALL 下添加一行内容
  bigdata ALL=(ALL)        NOPASSWD: ALL
  # 重新将文件设置为只读
  chmod u-w /etc/sudoers
  ```

  ![image-20210220112956256](/hadoop/创建普通用户-2.png)

  ![image-20210220113141520](/hadoop/创建普通用户-3.png)

  ![image-20210220113358317](/hadoop/创建普通用户-4.png)

  ![image-20210220113519585](/hadoop/创建普通用户-7.png)

### 重启虚拟机

- 为使配置生效，重启虚拟机：

  ```bash
  reboot
  ```

- 使用`bigdata`用户登录：

  ![image-20210220113841559](/hadoop/重启虚拟机-1.png)

- 验证虚拟机IP及主机名：

  ```bash
  # 查看当前主机IP
  ifconfig
  # 查看主机名
  hostname
  ```

  ![image-20210220114046296](/hadoop/重启虚拟机-2.png)

### 使用Xshell远程访问

在 VMware 上直接操作 Linux 用户体验不友好，故使用 `Xshell`来远程访问虚拟机。

-  修改Windows 主机映射文件（Win10 系统文件路径：C:\Windows\System32\drivers\etc）hosts,添加如下内容：

  ```
  192.168.1.101 bigdata101
  192.168.1.102 bigdata102
  192.168.1.103 bigdata103
  ```

![image-20210220114451191](/hadoop/使用Xshell远程访问-1.png)

![image-20210220114706174](/hadoop/使用Xshell远程访问.png)

- 使用`Xshell`新建会话，输入相关内容：

  ![image-20210220114902432](/hadoop/使用Xshell远程访问-3.png)

  ![image-20210220114953239](/hadoop/使用Xshell远程访问-4.png)

  ![image-20210220115027145](/hadoop/使用Xshell远程访问-5.png)

  ![image-20210220115109307](/hadoop/使用Xshell远程访问-8.png)

## 集群搭建

### 安装 JDK

- 在`bigdata101`虚拟机上创建目录，并调整目录所属用户及组

  ```bash
  sudo mkdir /opt/module /opt/software 
  ```

  ![image-20210222091426905](/hadoop/安装JDK-1.png)

  ![image-20210222092737377](/hadoop/安装JDK-2.png)

- 使用`Xftp`上传`JDK`安装包并解压

  ![image-20210222093306902](/hadoop/安装JDK-3.png)

  ![image-20210222093503660](/hadoop/安装JDK-4.png)

  ```bash
  cd /opt/software
  tar -zxvf jdk-8u212-linux-x64.tar.gz -C /opt/module/
  ```

  ![image-20210222093749934](/hadoop/安装JDK-5.png)

  ![image-20210222093851021](/hadoop/安装JDK-6.png)

- 配置环境变量，验证是否安装成功

  ```bash
  sudo touch /etc/profile.d/my_env.sh
  sudo vi /etc/profile.d/my_env.sh
  # 在my_env.sh输入以下内容后，保存退出
  export JAVA_HOME=/opt/module/jdk1.8.0_212
  export PATH=$PATH:$JAVA_HOME/bin
  ```

  ![image-20210222100602652](/hadoop/安装JDK-7.png)

  如图所示，打印出`Java`版本信息后即表明安装成功。

### 安装 Hadoop

- 将 `hadoop`安装包上传至 `/opt/software`目录 ；

- 解压安装

  ```bash
  cd /opt/software
  tar -zxvf hadoop-2.7.2.tar.gz -C /opt/module/
  ```

  ![image-20210222101919079](/hadoop/安装hadoop-1.png)

- 配置环境变量

  ```bash
  sudo vim /etc/profile.d/my_env.sh
  # 输入以下内容并保存退出
  #HADOOP_HOME
  export HADOOP_HOME=/opt/module/hadoop-2.7.2
  export PATH=$PATH:$HADOOP_HOME/bin
  export PATH=$PATH:$HADOOP_HOME/sbin
  ```

- 刷新环境变量，验证是否安装成功

  ```bash
  source /etc/profile.d/my_env.sh
  hadoop version
  ```

  ![image-20210222102342021](/hadoop/安装hadoop-2.png)

### 关闭防火墙

- 查看防火墙状态

  ```bash
  systemctl status firewalld.service
  ```

  ![image-20210222161409187](/hadoop/关闭防火墙-1.png)

- 关闭防火墙

  ```bash
  sudo systemctl stop firewalld.service
  ```

  ![image-20210222161606906](/hadoop/关闭防火墙-2.png)

- 永久关闭防火墙

  ```bash
  sudo systemctl disable firewalld.service
  ```

  ![image-20210222161706682](/hadoop/关闭防火墙-3.png)

### 其它节点虚拟机克隆

- 以 `bigdata-101`为模板，克隆两个虚拟机，命名为`bigdata-102`,`bigdata-103`(克隆前需要将`bigdata-101`进行关机操作)

- 修改对应节点虚拟机IP及主机名

  > bigdata-102 IP 修改为：192.168.1.102，主机名修改为：bigdata102
  >
  > bigdata-103 IP 修改为：192.168.1.103，主机名修改为：bigdata103

- 启动三台虚拟机

### 配置 SSH 无密钥登录

- 生成公钥和私钥

  ```bash
  # 执行后连敲三下空格
  ssh-keygen -t rsa
  ```

  ![image-20210222163846511](/hadoop/无密钥登录-1.png)

- 分发公钥

  ```bash
  ssh-copy-id bigdata101
  ssh-copy-id bigdata102
  ssh-copy-id bigdata103
  ```

  ![image-20210222164104210](/hadoop/无密钥登录-2.png)

  ![image-20210222164208551](/hadoop/无密钥登录-3.png)

  ![image-20210222164253884](/hadoop/无密钥登录-4.png)

- 切换到其它虚拟机分别执行生成公钥和私钥、分发公钥操作

- 切换`root`用户，对三台虚拟机分别执行生成公钥和私钥、分发公钥操作

- 验证 SSH 无密钥登录

  ```bash
  ssh bigdata101
  ssh bigdata102
  ssh bigdata103
  ```

  ![image-20210222164701608](/hadoop/无密钥登录-5.png)

### 编写文件集群分发脚本

- 在三台虚拟机上分别安装 `rsync` 服务

  ```bash
  sudo yum install -y rsync
  ```

  ![image-20210222165542853](/hadoop/分发脚本-1.png)

- 创建`/home/bigdata/bin`目录

  ```bash
  cd
  mkdir bin
  ```

  ![image-20210222165737043](/hadoop/分发脚本-2.png)

- 创建集群分发脚本

  ```bash
  cd /home/bigdata/bin
  touch xsync
  vi xsync
  ```

  ![image-20210222165908184](/hadoop/分发脚本-3.png)

- 输入脚本内容

  ```bash
  #!/bin/bash
  #1 获取输入参数个数，如果没有参数，直接退出
  pcount=$#
  if ((pcount==0)); then
  echo no args;
  exit;
  fi
  
  #2 获取文件名称
  p1=$1
  fname=`basename $p1`
  echo fname=$fname
  
  #3 获取上级目录到绝对路径
  pdir=`cd -P $(dirname $p1); pwd`
  echo pdir=$pdir
  
  #4 获取当前用户名称
  user=`whoami`
  
  #5 循环
  for host in bigdata101 bigdata102 bigdata103
  do
      echo ------------------- $host --------------
      rsync -av $pdir/$fname $user@$host:$pdir
  done
  ```

- 给脚本添加执行权限

  ```bash
  chmod u+x xsync
  ```

  ![image-20210222170145219](/hadoop/分发脚本-4.png)

### 修改Hadoop配置为集群配置

- HDFS 相关文件配置

  - 配置 core-site.xml

    ```bash
    <!-- 指定HDFS中NameNode的地址 -->
    <property>
    		<name>fs.defaultFS</name>
          <value>hdfs://bigdata101:9000</value>
    </property>
    
    <!-- 指定Hadoop运行时产生文件的存储目录 -->
    <property>
    		<name>hadoop.tmp.dir</name>
    		<value>/opt/module/hadoop-2.7.2/data/tmp</value>
    </property>
    ```

    注意：/opt/module/hadoop-2.7.2/data/tmp 需提前创建。

  - 配置 hadoop-env.sh

    ```bash
    export JAVA_HOME=/opt/module/jdk1.8.0_212
    ```

  - 配置 hdfs-site.xml

    ```bash
    <!-- 配置文件副本数 -->
    <property>
    	<name>dfs.replication</name>
    	<value>3</value>
    </property>
    <property>
        <name>dfs.http.address</name>
        <value>bigdata101:50070</value>
    </property>
    <!-- 指定Hadoop辅助名称节点主机配置 -->
    <property>
    	<name>dfs.namenode.secondary.http-address</name>
    	<value>bigdata103:50090</value>
    </property>
    ```

- YARN 文件配置

  - 配置yarn-site.xml

    ```bash
    <!-- Reducer获取数据的方式 -->
    <property>
    	<name>yarn.nodemanager.aux-services</name>
    	<value>mapreduce_shuffle</value>
    </property>
    
    <!-- 指定YARN的ResourceManager的地址 -->
    <property>
    	<name>yarn.resourcemanager.hostname</name>
    	<value>bigdata102</value>
    </property>
    
    <!-- 日志聚集功能使能 -->
    <property>
        <name>yarn.log-aggregation-enable</name>
        <value>true</value>
    </property>
    
    <!-- 日志保留时间设置7天 -->
    <property>
        <name>yarn.log-aggregation.retain-seconds</name>
        <value>604800</value>
    </property>
    ```

  - 配置 yarn-env.sh

    ```bash
    export JAVA_HOME=/opt/module/jdk1.8.0_212
    ```

- MapReduce 文件配置

  - 配置mapred-site.xml

    ```bash
    <!-- 指定MR运行在Yarn上 -->
    <property>
    	<name>mapreduce.framework.name</name>
    	<value>yarn</value>
    </property>
    
    <!-- 历史服务器端地址 -->
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>bigdata101:10020</value>
    </property>
    
    <!-- 历史服务器web端地址 -->
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>bigdata101:19888</value>
    </property>
    ```

  - 配置mapred-env.sh

    ```bash
    export JAVA_HOME=/opt/module/jdk1.8.0_212
    ```

- slaves 文件配置

  ```bash
  vi slaves
  # 添加以下内容
  bigdata101
  bigdata102
  bigdata103
  ```

- 将文件分发到其它节点

  ```bash
  xsync /opt/module/hadoop-2.7.2/
  ```

  ![image-20210222173922966](/hadoop/集群配置-1.png)

## 集群时间同步

- 使用`bigdata101`为时间服务，其它虚拟机时间跟它保持同步，切换`root`用户，在三台虚拟机上安装 `ntp` 服务

  ```bash
  su
  yum install -y ntp
  rpm -qa | grep ntp
  ```

  ![image-20210222174257588](/hadoop/集群时间同步-1.png)

- 修改 `ntp` 配置文件

  ```bash
  vi /etc/ntp.conf
  # 添加内容：授权192.168.1.0-192.168.1.255网段上的所有机器可以从这台机器上查询和同步时间
  restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap
  # 修改内容：集群在局域网中，不使用其他互联网上的时间
  #server 0.centos.pool.ntp.org iburst
  #server 1.centos.pool.ntp.org iburst
  #server 2.centos.pool.ntp.org iburst
  #server 3.centos.pool.ntp.org iburst
  # 添加内容：当该节点丢失网络连接，依然可以采用本地时间作为时间服务器为集群中的其他节点提供时间同步
  server 127.127.1.0
  fudge 127.127.1.0 stratum 10
  ```

  ![image-20210222175105073](/hadoop/时间同步-2.png)

- 修改/etc/sysconfig/ntpd 文件

  ```bash
  vim /etc/sysconfig/ntpd
  # 添加以下内容：让硬件时间与系统时间一起同步
  SYNC_HWCLOCK=yes
  ```

- 重新启动`ntpd`服务并设置开机启动

  ```bash
  # 启动服务
  service ntpd start
  # 查看服务状态
  service ntpd status
  # 设置开机启动
  chkconfig ntpd on
  ```

  ![image-20210222175602505](/hadoop/时间同步-3.png)

- 其它机器配置（必须使用`root`用户）

  - 在其他机器配置10分钟与时间服务器同步一次

    ```bash
    crontab -e
    # 编写定时任务如下
    */10 * * * * /usr/sbin/ntpdate bigdata101
    ```

    