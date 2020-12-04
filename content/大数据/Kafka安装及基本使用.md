---
date: 2020-11-11
title: "Kafka安装及基本使用"
tags: ["大数据", "Kafka","中间件"]
---

## 安装前准备

安装 Kafka 依赖于 ZooKeeper，所以需要提前将 ZooKeeper 服务启动起来。

## 安装过程

- 安装包下载

  - 确定 scala 版本：

    ```bash
    scala -version
    # 以下为输出内容
    Scala code runner version 2.11.6 -- Copyright 2002-2013, LAMP/EPFL
    ```

  - 下载对应 scala 版本的安装包，[下载链接](http://kafka.apache.org/downloads)。

- 添加执行权限并解压

  ```bash
  # 添加权限
  chmod u+x kafka_2.11-2.4.1.tgz 
  # 解压
   tar -zxf kafka_2.11-2.4.1.tgz -C /usr/local/
  # 重命名
  mv kafka_2.11-2.4.1/ kafka
  ```

- 修改配置文件

  - 修改 server.properties

    ```bash
    vi server.properties 
    # 以下为修改的内容
    # 设置 brokerId,每个节点的 brokerId都要是全局唯一
    broker.id=0
    # 设置监听的主机名：端口
    listeners=PLAINTEXT://master:9092
    
    # 设置日志文件存储路径
    log.dirs=/usr/local/kafka/kafka-logs
    # 配置连接的 ZooKeeper 地址
    zookeeper.connect=zoo1:2181,zoo2:2181,zoo3:2181
    
    ```

- 将安装文件发送到其它节点后，再修改对应节点的配置文件

  ```bash
  scp -r /usr/local/kafka/ slave1:/usr/local/
  scp -r /usr/local/kafka/ slave2:/usr/local/
  ```

- 启动服务

  ```bash
  cd /usr/local/kafka/
  # 启动服务
  bin/kafka-server-start.sh config/server.properties &
  ```

- 功能验证

  - 创建主题

  ```bash
  bin/kafka-topics.sh --create --topic quickstart-events --bootstrap-server master:9092
  ```

  - 在 master 节点启动生产者

    ```bash
    bin/kafka-console-producer.sh --topic quickstart-events --broker-list master:9092,slave1:9092,slave2:9092
    ```

  - 在 slave1 节点启动消费者

    ```bash
    bin/kafka-console-consumer.sh --topic quickstart-events --from-beginning --bootstrap-server slave1:9092
    ```

    在 master 节点终端上输入字符，可以看到 slave1 节点终端能即时输出，说明功能正常。

## 相关概念