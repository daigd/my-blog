---
date: 2020-11-11
title: "Hive安装及基本使用"
tags: ["大数据", "Hive安装"]安装
---

## 安装前准备

Hive 只是一个方便使用 Hadoop mapreduce 功能的客户端工具，因此安装 Hive 前需要先安装好 Hadoop,基于[使用Docker构建Hadoop集群](https://vigorous-wozniak-4b6bd2.netlify.app/%E5%A4%A7%E6%95%B0%E6%8D%AE/%E4%BD%BF%E7%94%A8docker%E6%9E%84%E5%BB%BAhadoop%E9%9B%86%E7%BE%A4/),假设 Hadoop 集群已安装好。

## 安装过程

- 进入 Hadoop 集群的 master 节点，切换到 /root 目录，下载 Hive 安装包并解压，(具体下载版本可参考[官网](https://hive.apache.org/downloads.html))：

```bash
# 进入 master 容器
docker exec -it master /bin/bash
# 切换目录并下载安装包
cd /root/
wget https://mirrors.tuna.tsinghua.edu.cn/apache/hive/hive-2.3.7/apache-hive-2.3.7-bin.tar.gz
# 解压至指定路径
tar -zxvf apache-hive-2.3.7-bin.tar.gz -C /usr/local/
# 切换目录，对解压后的 hive 目录重命名
cd /usr/local/
mv apache-hive-2.3.7-bin/ hive 
```

- 修改配置文件

  进入解压后的目录，找到 conf 目录，修改配置文件

  ```bash
  cd hive/conf/
  # 如果没有 hive-env.sh 文件，cp ./hive-env.sh.template hive-env.sh
  vi hive-env.sh
  # 设置 hadoop 安装目录
  HADOOP_HOME=/usr/local/hadoop
  ```

- 配置环境变量

  ```bash
  vi /etc/profile
  # 文件末追加下面内容
  #hive
  export HIVE_HOME=/usr/local/hive  
  export PATH=${HIVE_HOME}/bin:$PATH
  ```

  修改完后执行 source /etc/profile 使配置内容立即生效。

- 元数据存储配置

  Hive 元数据存储有两种方式：derby 和 MySql。

  | 存储方式   | 优点                           | 缺点                                       |
  | ---------- | ------------------------------ | ------------------------------------------ |
  | 内置 derby | bin/hive 启动即可使用,简单方便 | 不同路径启动都会有一套元数据版本，无法共享 |
  | MySQL 方式 | 元数据共享，管理方便           | 依赖MySQL                                  |

  生产环境上一般是使用 MySQL 方式来管理 Hive 元数据，故下面介绍使用 MySQL 来管理 Hive 元数据。

  - 上传 MySQL 驱动包到 （如mysql-connector-java-5.1.49.jar）hive 安装目录的 lib 目录下

    [MySQL 驱动包下载页面](https://dev.mysql.com/downloads/connector/j/5.1.html)

  - 修改配置文件（假设已安装好MqSQL，并创建一个名为 hive 的数据库）

    ```bash
    cd /usr/local/hive/conf/
    # 如果没有该配置文件,执行 cp hive-default.xml.template hive-site.xml 
    vi hive-site.xml  
    ```

    属性修改：

    ```xml
      <property>
          <!--指定MySQL用户名-->
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
        <description>Username to use against metastore database</description>
      </property>
      <property>
          <!--指定MySQL密码-->
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>root</value>
        <description>password to use against metastore database</description>
      </property>
      <property>
          <!--指定MySQL地址：mysql-01对应MySQLIP-->
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://mysql-01:3306/hive?createDatabaseIfNotExist=true</value>
      </property>
      <property>
          <!--指定数据库驱动-->
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
        <description>Driver class name for a JDBC metastore</description>
      </property>
    ```

    属性添加：

    ```xml
      <property>
        <name>system:java.io.tmpdir</name>
          <!--需提前创建该目录-->
        <value>/root/hive/tmp</value>
      </property>
      <property>
        <name>system:user.name</name>
        <value>${user.name}</value>
      </property>
    ```

    至此，MySQL 配置完毕。

- 启动 Hive

  - 数据库初始化

    ```bash
    schematool -dbType mysql -initSchema
    ```

  - 启动 Hive 的 metastore 元数据服务

    ```bash
    hive --service metastore
    ```

  - 启动 Hive

    ```bash
    hive
    ```

## Hive 基本使用

首先执行 hive 进行命令行窗口。

- 创建数据库

  ```bash
  CREATE DATABASE test;
  ```

- 显示所有数据库

  ```bash
  show DATABASES;
  ```

- 创建表

  ```bash
  use test;
  CREATE TABLE student (classNo string,strNo string,score int) row format delimited fields terminated by ',';
  ```

  - row format delimited fields terminated by ',' 指定了字段的分隔符为逗号，所以导入数据的时候文本也要为逗号，否则加载后数据为 NULL。
  - Hive 只支持单个字符的分隔符，默认的分隔符是 \001。

- 显示所有的表

  ```bash
  show TABLES;
  ```

- 将数据导入到表中

  创建一个测试文本：

  ```bash
  echo "C01,NO101,89
  C01,NO102,87
  C01,NO103,94
  C01,NO104,75
  C01,NO105,80" > /root/student.txt
  ```

  导入数据，在Hive命令行窗口执行下列命令：

  ```bash
  load data local inpath '/root/student.txt' overwrite into table student;
  ```

  这个命令将目标文件复制到 hive 的 warehouse 目录中，这个目录由 hive-site.xml 配置文件中 hive.metastore.warehouse.dir 配置项决定，默认值为 /user/hive/warehouse。overwrite 选项将导致 Hive 事先删除 student 目录下所有的文件，并将文件内容映射到表中。

- 数据查询，用法跟 SQL 类似。

  基本查询：

  ```sql
  use test;
  select * from student;
  ```

   分组查询 group by 和统计 count：

  ```sql
  use test;
  select classNo,count(score) from student where score>80 group by classNo;
  ```

  从执行结果可以看，Hive 把查询变成了 MapReduce 作业通过 Hadoop 来执行。

  （Hive 的其它使用后续再更新。）