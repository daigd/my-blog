---
date: 2020-11-11
title: "HBase安装及基本使用"
tags: ["大数据", "HBase基本使用"]
---

## 安装前准备

安装 HBase 依赖于 ZooKeeper 及 HDFS，所以需要提前将 ZooKeeper 及 Hadoop 的 DFS 服务启动起来。

## 安装过程

- 下载安装

  这里以 cdh 版本来介绍其安装过程，首先下载对应安装包(以cdh5.9.3版本为例)，[链接](http://archive.cloudera.com/cdh5/cdh/5/)，下载成功后更改权限及解压：

  ```bash
  # 设置可执行权限
  chmod u+x hbase-1.2.0-cdh5.9.3.tar.gz
  # 解压
  tar -zxf hbase-1.2.0-cdh5.9.3.tar.gz -C /usr/local
  # 目录重命名
  mv hbase-1.2.0-cdh5.9.3/ hbase
  ```

- 修改配置文件

  ```bash
  # 进入配置文件目录
  cd hbase/conf/
  ```

  - 修改 hbase-env.sh 文件：

    ```bash
    vi hbase-env.sh 
    # 设置 JAVA_HOME
    export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
    # 不使用内置的ZooKeeper服务
    export HBASE_MANAGES_ZK=false
    ```

  - 修改 hbase-site.xml 文件：

    ```bash
    vi hbase-site.xml 
    ```

    添加如下配置,其它参数可参考[官网](https://hbase.apache.org/book.html#config.files)：

    hbase.rootdir : region server 的共享目录，用来持久化 HBase，没更改默认配置的话，数据会在重启后丢失。

    hbase.cluster.distributed：是否为集群模式。

    hbase.zookeeper.quorum：监听 ZooKeeper 服务的主机名，多个 ZK 服务以英文逗号分隔。

    ```bash
    <configuration>
      <property>
        <name>hbase.rootdir</name>
        <value>hdfs://master:9000/hbase</value>
      </property>
      <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
      </property>
      <property>
        <name>hbase.zookeeper.quorum</name>
        <value>zoo1,zoo2,zoo3</value>
      </property>
    </configuration>
    ```

  - 配置 region server 节点：

    ```bash
    vi regionservers 
    # 配置 region server 节点主机名
    master
    slave1
    slave2
    ```

- 启动服务

  - 将修改好的配置文件分发到 slave1和slave2 节点 /usr/local/ 中：

    ```bash
    scp -r /usr/local/hbase/ slave1:/usr/local/
    scp -r /usr/local/hbase/ slave2:/usr/local/
    ```

  - 执行启动脚本：

    ```bash
     ./start-hbase.sh 
    ```

  - 进入 HBase shell 命令行模式：

    ```bash
    ./hbase shell
    ```

    看到输出以下内容，则证明 HBase 服务启动成功了：

    ```bash
    2020-12-04 03:33:25,104 INFO  [main] Configuration.deprecation: hadoop.native.lib is deprecated. Instead, use io.native.lib.available
    2020-12-04 03:33:27,123 WARN  [main] util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
    HBase Shell; enter 'help<RETURN>' for list of supported commands.
    Type "exit<RETURN>" to leave the HBase Shell
    Version 1.2.0-cdh5.9.3, rUnknown, Tue Jun 27 16:19:14 PDT 2017
    hbase(main):001:0> 
    ```
## 相关概念

## 基本使用

- 查看当前表

  命令：list

  示例：

  ```bash
  hbase(main):001:0> list
  TABLE                                                                                                                                                                                                                                           
  0 row(s) in 0.2740 seconds
  => []
  ```

- 创建表

  create '表名','列簇名1','列簇名2’

  示例：

  ```bash
  hbase(main):002:0> create 'test','info','info2'
  0 row(s) in 28.4680 seconds
  
  => Hbase::Table - test
  ```

- 添加记录

  put '表名‘,'rowKey','列簇名:列名','列对应的值'

  示例：

  ```bash
  put 'test','001','info:age','29'
  ```

- 查询记录（HBase 只能根据 RowKey 来查找记录）

  get '表名'，'rowKey'

  示例：

  ```bash
  hbase(main):005:0> get 'test','001'
  COLUMN                                                        CELL                                                                                                                                                                          
   info:age                                                    timestamp=1607060951036, value=29                                                                                                                                                  
   info:name                                                   timestamp=1607060645893, value=dgd                                                                                                                                                 
   info2:name                                                   timestamp=1607060663840, value=dgd2     
  ```

- 删除表

  删除表前要先禁止表，命令：

   disable '表名'

   drop '表名'

  示例：

  ```bash
   disable 'test'
   drop 'test'
  ```

  