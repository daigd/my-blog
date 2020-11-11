---
date: 2020-11-09
title: "使用Docker构建Hadoop集群"
tags: ["大数据", "Hadoop环境搭建"]
---

## Docker环境安装（略）

## 构建Hadoop集群

### 基础组件准备

基于ubuntu系统来搭建Hadoop集群，拉取操作系统镜像：

```bash
docker pull ubuntu:16.04
```

运行镜像并进入到操作系统：

```bash
docker run -it --name "ubuntu01" ubuntu:16.04 /bin/bash
```

对ubuntu自带的apt源进行备份，然后将apt源配置改为阿里的源，这样安装工具下载速度会比较快：

- apt源文件备份

  ```bash
  cp /etc/apt/sources.list /etc/apt/sources.list_back
  ```

- 切换阿里的源

  ```bash
  echo "deb http://mirrors.aliyun.com/ubuntu/ xenial main
       deb-src http://mirrors.aliyun.com/ubuntu/ xenial main
  
       deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main
       deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main
  
       deb http://mirrors.aliyun.com/ubuntu/ xenial universe
       deb-src http://mirrors.aliyun.com/ubuntu/ xenial universe
       deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
       deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
  
       deb http://mirrors.aliyun.com/ubuntu/ xenial-security main
       deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main
       deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
       deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security universe" > /etc/apt/sources.list
  ```

安装必要软件

- 更新系统软件

  ```bash
  apt update
  ```

- 安装 jdk（版本>=8）,scala，python

  ```bash
  apt install openjdk-8-jdk
  apt install scala
  apt install python3
  ```

- 安装编辑工具vim和网络工具 net-tools

  ```bash
   apt install vim
   apt install net-tools
  ```

- 安装ssh服务端和客户端，用于远程免密访问

  ```bash
  apt install openssh-server
  apt install openssh-client
  ```

  进入当前用户目录，然后生成免密登录的key。生成key的时候直接按回车即可。

  ```bash
  cd
  ssh-keygen -t rsa -P ""
  ```

  生成的 key 结果类似如下：

  ```bash
  Generating public/private rsa key pair.
  Enter file in which to save the key (/root/.ssh/id_rsa): 
  Created directory '/root/.ssh'.
  Your identification has been saved in /root/.ssh/id_rsa.
  Your public key has been saved in /root/.ssh/id_rsa.pub.
  The key fingerprint is:
  SHA256:hMY9yd5akSxQwGs3ha4BAv/Otx6foRFGIElmOQ+H3hc root@3500f09b4bf9
  The key's randomart image is:
  +---[RSA 2048]----+
  |.o=+..oo..       |
  | +B.ooE=.o..     |
  | ..B .=+B.+      |
  |  ..oo=o++ .     |
  |    .oo+S.o      |
  |   o ... o       |
  |    o + o        |
  |     . * o       |
  |     .+ o        |
  +----[SHA256]-----+
  ```

  这个时候在当前目录下会生成一个.ssh文件夹：

  ```bash
  root@3500f09b4bf9:~# ls -al
  total 24
  drwx------  3 root root 4096 Nov  9 07:12 .
  drwxr-xr-x 42 root root 4096 Nov  9 07:10 ..
  -rw-r--r--  1 root root 3106 Oct 22  2015 .bashrc
  -rw-r--r--  1 root root  148 Aug 17  2015 .profile
  -rw-------  1 root root   30 Nov  9 07:09 .python_history
  drwx------  2 root root 4096 Nov  9 07:12 .ssh
  ```

  将生成的公钥 key 追加到授权的 key 列表：

  ```bash
  cat .ssh/id_rsa.pub >> .ssh/authorized_keys
  ```

  启动ssh服务，然后进行本机测试，如果能成功登陆，说明ssh 无密登录已经成功了：

  ```bash
  service ssh start
  ssh 127.0.0.1
  ```

  为了保证系统启动的时候也启动ssh服务，我们将启动命令 “service ssh start” 追加到 bashrc 文件末尾中：

  ```bash
  vim ~/.bashrc
  ```

### 制作Hadoop镜像

接下来，开始下载 hadoop，将文件进行解压放到 /usr/local，并改名为 hadoop。

- 下载链接

  - http://archive.cloudera.com/cdh5/cdh/5/

    > cdh5基于官网整合了hadoop相关资源的版本差异性,用统一的cdh版本号来管理各个资源,优先使用该链接下载（缺点就是版本更新比较慢）。

  - http://hadoop.apache.org/

    > 官网下载链接。

#### Hadoop 2.X版本

```bash
wget http://archive.cloudera.com/cdh5/cdh/5/hadoop-2.6.0-cdh5.9.3.tar.gz
```

下载成功后，解压，重命名：

```bash
tar -zxvf hadoop-2.6.0-cdh5.9.3.tar.gz -C /usr/local
cd /usr/local
mv hadoop-2.6.0-cdh5.9.3 hadoop
```

配置 Java 和 Hadoop 相关环境变量

- 查看 Java 安装路径

  ```bash
  update-alternatives --config java
  ```

- 配置 Java、Hadoop 安装路径

  ```bash
  vim /etc/profile
  # 输入以下内容
  #java
  export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
  export JRE_HOME=${JAVA_HOME}/jre    
  export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib    
  export PATH=${JAVA_HOME}/bin:$PATH
  #hadoop
  export HADOOP_HOME=/usr/local/hadoop
  export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
  export HADOOP_COMMON_HOME=$HADOOP_HOME 
  export HADOOP_HDFS_HOME=$HADOOP_HOME 
  export HADOOP_MAPRED_HOME=$HADOOP_HOME
  export HADOOP_YARN_HOME=$HADOOP_HOME 
  export HADOOP_INSTALL=$HADOOP_HOME 
  export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native 
  export HADOOP_LIBEXEC_DIR=$HADOOP_HOME/libexec 
  export JAVA_LIBRARY_PATH=$HADOOP_HOME/lib/native:$JAVA_LIBRARY_PATH
  export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
  export HDFS_DATANODE_USER=root
  export HDFS_DATANODE_SECURE_USER=root
  export HDFS_SECONDARYNAMENODE_USER=root
  export HDFS_NAMENODE_USER=root
  export YARN_RESOURCEMANAGER_USER=root
  export YARN_NODEMANAGER_USER=root
  ```

  执行“source /etc/profile”，让配置的变量内容立即成生效。

- 修改 Hadoop 的环境变量

  ```bash
  vim /usr/local/hadoop/etc/hadoop/hadoop-env.sh
  # 输入以下内容
  export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
  export HDFS_NAMENODE_USER=root
  export HDFS_DATANODE_USER=root
  export HDFS_SECONDARYNAMENODE_USER=root
  export YARN_RESOURCEMANAGER_USER=root
  export YARN_NODEMANAGER_USER=root   
  ```

- 修改  Hadoop 相关配置文件

  修改 core-site.xml（路径为： /usr/local/hadoop/etc/hadoop），加入以下内容。fs.default.name 为默认的master节点。hadoop.tmp.dir 为hadoop默认的文件路径。如果本机没有的话需要自己通过 mkdir 命令进行创建。（更多参数可参阅[官网](http://archive.cloudera.com/cdh5/cdh/5/hadoop-2.6.0-cdh5.9.3/hadoop-project-dist/hadoop-common/core-default.xml)）

  ```bash
  <configuration>
      <property>
          <name>fs.default.name</name>
          <value>hdfs://master:9000</value>
      </property>
      <property>
          <name>hadoop.tmp.dir</name>
          <value>/root/hadoop/tmp</value>
      </property>
  </configuration>
  ```

  修改同目录下的 hdfs-site.xml。dfs.replication即从节点的数量，2个从节点。dfs.namenode.name.dir为 namenode 的元数据存放的路径，dfs.datanode.data.dir是数据存放的路径。（更多参数及参数含义可参阅[官网](http://archive.cloudera.com/cdh5/cdh/5/hadoop-2.6.0-cdh5.9.3/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml)）

  ```xml
  <configuration>
      <property>
          <name>dfs.replication</name>
          <value>2</value>
      </property>
      <property>
          <name>dfs.namenode.name.dir</name>
          <value>/root/hadoop/hdfs/name</value>
      </property>
      <property>
          <name>dfs.datanode.data.dir</name>
          <value>/root/hadoop/hdfs/data</value>
      </property>
      <property>
        <name>dfs.http.address</name>
        <value>master:9870</value>
  	</property>
  </configuration>
  ```

  修改同目录下的mapred-site.xml（如果没有，将 mapred-site.xml.template 复制一份并命名为 mapred-site.xml）。指定mapreduce执行框架设置为yarn。mapreduce.application.classpath 设置mapreduce的应用执行类路径。（更多参数及参数含义可参阅[官网](http://archive.cloudera.com/cdh5/cdh/5/hadoop-2.6.0-cdh5.9.3/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml)）

  ```xml
  <configuration>
      <property>
          <name>mapreduce.framework.name</name>
          <value>yarn</value>
      </property>
      <property>
          <name>mapreduce.application.classpath</name>
          <value>
              /usr/local/hadoop/etc/hadoop,
              /usr/local/hadoop/share/hadoop/common/*,
              /usr/local/hadoop/share/hadoop/common/lib/*,
              /usr/local/hadoop/share/hadoop/hdfs/*,
              /usr/local/hadoop/share/hadoop/hdfs/lib/*,
              /usr/local/hadoop/share/hadoop/mapreduce/*,
              /usr/local/hadoop/share/hadoop/mapreduce/lib/*,
              /usr/local/hadoop/share/hadoop/yarn/*,
              /usr/local/hadoop/share/hadoop/yarn/lib/*
          </value>
      </property>
  </configuration>
  ```

  修改同目录下的 yarn-site.xml 。yarn.resourcemanager.hostname为yarn资源管理的主机名为master机器。 yarn.nodemanager.aux-services说明当前yarn采用mapreduce洗牌的方式进行处理。（更多参数及参数含义可参阅[官网](http://archive.cloudera.com/cdh5/cdh/5/hadoop-2.6.0-cdh5.9.3/hadoop-yarn/hadoop-yarn-common/yarn-default.xml)）

  ```xml
  <configuration>
      <property>
          <name>yarn.resourcemanager.hostname</name>
          <value>master</value>
      </property>
      <property>
          <name>yarn.nodemanager.aux-services</name>
          <value>mapreduce_shuffle</value>
      </property>
      <!--物理内存配置-->
      <property>
  		<name>yarn.nodemanager.resource.memory-mb</name>
  		<value>2048</value>
  	</property>
      <!--最低内存配置-->
      <property>
  		<name>yarn.scheduler.minimum-allocation-mb</name>
  		<value>1024</value>
  	</property>
      <property>
  		<name>yarn.nodemanager.disk-health-checker.max-disk-utilization-per-disk-percentage</name>
  		<value>98.5</value>
  	</property>    
  </configuration>
  ```

  修改同目录下的 slaves 文件（Hadoop 3.X 版本文件名换成了 workers）, 即告诉Hadoop worker的机器为哪些。（默认为localhost，单机版可不用修改），加入如下内容。

  ```sh
  slave1
  slave2
  ```

  进入Hadoop的bin路径下将namenode进行格式化。这样hadoop才可以使用，否则无法使用。

  ```bash
  ./bin/hadoop namenode -format
  ```

  Hadoop的基本配置文件和内容已经配置好，下一步我们退出 docker 容器，Ctrl+c 或者输入exit即可退出。我们通过docker ps -a 查看这个进行的container id。接下来保存为 ubuntu:hadoop-2.6.0 镜像。

  ```bash
  docker commit -m "hadoop 2.6.0" {CONTAINER ID} ubuntu:hadoop-2.6.0
  ```

#### Hadoop 3.X版本

从官网上下载 3.X 版本:

```bash
wget https://downloads.apache.org/hadoop/common/hadoop-3.3.0/hadoop-3.3.0.tar.gz 
```

下载成功后，解压，重命名：

```bash
tar -zxvf hadoop-3.3.0.tar.gz -C /usr/local
cd /usr/local
mv hadoop-3.3.0/ hadoop
```

配置 Java 和 Hadoop 相关环境变量（参考2.X版本）（详情可参阅[官网](https://hadoop.apache.org/docs/r3.3.0/hadoop-project-dist/hadoop-common/SingleCluster.html)）。

core-site.xml 添加内容调整（更多参阅[官网](https://hadoop.apache.org/docs/r3.3.0/hadoop-project-dist/hadoop-common/core-default.xml)）

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://master:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/root/hadoop/tmp</value>
    </property>
</configuration>
```

保存镜像：

```bash
docker commit -m "hadoop 3.3.0" {CONTAINER ID} ubuntu:hadoop-3.3.0
```

### 容器部署

创建自定义网络

```bash
docker network create --driver=bridge hadoop
docker network ls
```

打开三个终端容器，依次启动三个容器（-h 指定主机名，有了主机名可以直接通过主机名访问机器），如果只配置单机版，只启动 master 节点即可：

```bash
docker run -it --network hadoop -h "master" --name "master" -p 9870:9870 -p 8088:8088 ubuntu:hadoop-2.6.0 /bin/bash
docker run -it --network hadoop -h "slave1" --name "slave1" ubuntu:hadoop-2.6.0 /bin/bash
docker run -it --network hadoop -h "slave2" --name "slave2" ubuntu:hadoop-2.6.0 /bin/bash
```

进入 master 容器，启动所有 hadoop 节点：

```bash
cd /usr/local/hadoop/sbin/
./start-all.sh
```

如果一切启动正常的话，访问本地 localhost:9870 和 localhost:8088 （如果不是本地搭建，localhost 换成对应主机 ip 地址）,页面分别如下：

![](/hadoop/hadoop-index.png)

![](/hadoop/yanr-index.png)

### 功能验证

使用内置的 wordcount 程序来验证一下 Hadoop 功能是否正常。

进入 master 容器，在 /root 目录下创建测试文件 test.txt：

```bash
docker exec -it master /bin/bash
echo "Java Go
Spark Hive
HBase Java
Hadoop Spark" > /root/test.txt
```

进入 Hadoop的 bin 目录，  在HDFS 中创建一个 /input 目录，然后将测试文件上传到 /input 目录中，检查是否上传成功，接着就可以调用内置的 wordcount 程序，将结果写入到 /output 目录。

```bash
cd /usr/local/hadoop/bin
# 在HDFS创建/input目录
./hadoop fs -mkdir /input
# 上传文件
./hadoop fs -put /root/test.txt /input
# 检查是否上传成功
./hadoop fs -ls /input
./hadoop fs -cat /input/test.txt
# 使用内置 wordcount 程序
./hadoop jar ../share/hadoop/mapreduce/hadoop-mapreduce-examples-2.6.0-cdh5.9.3.jar wordcount /input/test.txt /output
```

执行完毕后，如果出现如下结果说明 mapreduce 程序执行成功：

```bash
# 忽略
.....
20/11/11 01:52:25 INFO mapreduce.Job: Running job: job_1605059372659_0001
20/11/11 01:52:31 INFO mapreduce.Job: Job job_1605059372659_0001 running in uber mode : false
20/11/11 01:52:31 INFO mapreduce.Job:  map 0% reduce 0%
20/11/11 01:52:38 INFO mapreduce.Job:  map 100% reduce 0%
20/11/11 01:52:43 INFO mapreduce.Job:  map 100% reduce 100%
20/11/11 01:52:43 INFO mapreduce.Job: Job job_1605059372659_0001 completed successfully
# 忽略
......
```

查看下输出结果：

```bash
./hadoop fs -ls /output
# 下面为输出结果内容
20/11/11 01:59:13 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Found 2 items
-rw-r--r--   2 root supergroup          0 2020-11-11 01:52 /output/_SUCCESS
-rw-r--r--   2 root supergroup         44 2020-11-11 01:52 /output/part-r-00000
```

可以看到输出目录下有两个文件，_SUCCESS 表明程序已经执行成功，part-r-00000 是输出结果集，查看下里面内容：

```bash
./hadoop fs -text /output/part-r-00000
# 下面为显示内容
20/11/11 02:03:08 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Go	1
HBase	1
Hadoop	1
Hive	1
Java	2
Spark	2
```

可以看到输出的统计结果跟我们输入的内容一致，至此也说明了 mapreduce 功能正常。

### 文档参考

[使用docker构建hadoop集群](https://www.cnblogs.com/huangqingshi/p/12531717.html)