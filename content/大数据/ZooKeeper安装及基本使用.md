---
date: 2020-11-11
title: "ZooKeeper安装及基本使用"
tags: ["大数据", "ZooKeeper安装"]
---

## 简要介绍

## 集群安装

ZooKeeper 集群安装使用  docker-compose 方式([参考](https://hub.docker.com/_/zookeeper))：

- 创建 compose 文件，命名为zk-cluster.yml，输入以下内容：

  ```yaml
  version: '3.1'
  
  services:
    zoo1:
      image: zookeeper:3.5
      container_name: zoo1
      restart: always
      hostname: zoo1
      ports:
        - 2181:2181
      environment:
        ZOO_MY_ID: 1
        ZOO_SERVERS: server.1=0.0.0.0:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181
      networks:
        - hadoop
        
    zoo2:
      image: zookeeper:3.5
      container_name: zoo2
      restart: always
      hostname: zoo2
      ports:
        - 2182:2181
      environment:
        ZOO_MY_ID: 2
        ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=0.0.0.0:2888:3888;2181 server.3=zoo3:2888:3888;2181
      networks:
        - hadoop
        
    zoo3:
      image: zookeeper:3.5
      container_name: zoo3
      restart: always
      hostname: zoo3
      ports:
        - 2183:2181
      environment:
        ZOO_MY_ID: 3
        ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=0.0.0.0:2888:3888;2181
      networks:
        - hadoop
        
  networks:
    hadoop:
      external: true
  ```

  注意：需要提前创建名为 hadoop 的 docker 自定义网络。

- 执行以下命令：

  ```bash
  docker-compose -p zk -f ./zk-cluster.yml up -d
  ```

- 启动成功后，进入 zoo1 容器验证一下：

  ```bash
  # 进入zoo1容器
  docker exec -it zoo1 /bin/bash
  # 查看服务状态
  ./bin/zkServer.sh status
  ```

  显示以下内容，说明服务启动正常，且当前节点角色为 follower:

  ```bash
  ZooKeeper JMX enabled by default
  Using config: /conf/zoo.cfg
  Client port found: 2181. Client address: localhost.
  Mode: follower
  ```

## 基本使用
- 连接到 ZooKeeper：

  ```bash
  bin/zkCli.sh -server zoo1:2181,zoo2:2182,zoo3:2183
  ```

- 执行 help 命令查看有哪些命令可用：

  ```bash
  [zk: zoo1:2181,zoo2:2182,zoo3:2183(CONNECTED) 0] help
  ```

  显示内容如下：

  ```bash
  ZooKeeper -server host:port cmd args
  	addauth scheme auth
  	close 
  	config [-c] [-w] [-s]
  	connect host:port
  	create [-s] [-e] [-c] [-t ttl] path [data] [acl]
  	delete [-v version] path
  	deleteall path
  	delquota [-n|-b] path
  	get [-s] [-w] path
  	getAcl [-s] path
  	history 
  	listquota path
  	ls [-s] [-w] [-R] path
  	ls2 path [watch]
  	printwatches on|off
  	quit 
  	reconfig [-s] [-v version] [[-file path] | [-members serverID=host:port1:port2;port3[,...]*]] | [-add serverId=host:port1:port2;port3[,...]]* [-remove serverId[,...]*]
  	redo cmdno
  	removewatches path [-c|-d|-a] [-l]
  	rmr path
  	set [-s] [-v version] path data
  	setAcl [-s] [-v version] [-R] path acl
  	setquota -n|-b val path
  	stat [-w] path
  	sync path
  ```

  