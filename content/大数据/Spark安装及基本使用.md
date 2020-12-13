---
date: 2020-12-13
title: "Spark安装及基本使用"
tags: ["大数据", "Spark"]
---

## 安装

- 安装 JDK 及 Scala（全部软件都默认装在Linux操作系统上），本次安装的 Spark 版本为 2.4.7，JDK 版本要求为8，Scala 版本要求为 2.11，各版本要求可查看[官网](http://spark.apache.org/documentation.html)

- 安装 Spark,下载地址：[spark-2.4.7-bin-hadoop2.7.tgz](https://www.apache.org/dyn/closer.lua/spark/spark-2.4.7/spark-2.4.7-bin-hadoop2.7.tgz)。

  - 解压至指定目录：

    ```bash
    tar -zxf spark-2.4.7-bin-hadoop2.7.tgz -C /usr/local
    ```

    

## 基本使用

### 本地模式

方式一：通过 spark-shell 方式：

- 进入 bin 目录：

  ```bash
  cd /usr/local/spark-2.4.7-bin-hadoop2.7
  ```

- 通过 spark-shell 方式启动:

  ```bash
  bin/spark-shell --master local[2]
  ```

  > master参数后面使用 `local`指定为本地模式，2 表示启用两个线程来进行计算。

- 输入 :paster ，就可以输入一段代码，然后按 ctrl+D 完成代码输入:

  ```scala
  val textFile = spark.read.textFile("file:///usr/local/spark-2.4.7-bin-hadoop2.7/README.md")
  val numAs = textFile.filter(_.contains("a")).count()
  val numBs = textFile.filter(_.contains("Spark")).count()
  println(s"Lines with a: $numAs, Lines with b: $numBs")
  ```

  结果输出如下：

  ```bash
  Lines with a: 61, Lines with b: 19
  textFile: org.apache.spark.sql.Dataset[String] = [value: string]
  numAs: Long = 61
  numBs: Long = 19
  ```

方式二：生成 jar 包并提交 scala-submit 运行：

- 在 IDE 上创建一个Scala 对象，输入以下代码([Github](https://github.com/daigd/StudyDemo/blob/master/spark-demo/src/main/scala/SparkDemo.scala))：

  ```scala
  object SparkDemo {
    def main(args: Array[String]): Unit = {
      // 通过命令行参数指定文件路径
      if (args.isEmpty||args(0).isEmpty) {
        throw new Exception("File path cannot be null!")
      }
      val textPath = args(0)
      val spark = SparkSession.builder().appName("app").config("spark.master", "local")
        .getOrCreate()
      val textFile = spark.read.textFile(textPath)
      val numA = textFile.filter(_.contains("a")).count()
      val numSpark = textFile.filter(_.contains("Spark")).count()
      println(s"Lines contains a: $numA, Lines contains Spark: $numSpark")
      spark.stop()
    }
  }
  ```

- 代码打成 jar 后上传到服务器上，放至 /root 目录中，执行以下命令：

  ```bash
  cd /usr/local/spark-2.4.7-bin-hadoop2.7/
  # bin/spark-submit --master local[2] jar路径 参数
  bin/spark-submit --master local[2] /root/spark-demo.jar file:///root/test.txt
  ```

  注意：需要提前在 /root 目录下创建 test.txt 文件，内容如下：

  ```bash
  Java Hadoop
  Spark Hive
  HBase Go
  Java Spark
  Scala Hadoop
  ```

  程序执行完毕输出以下内容：

  ```bash
  Lines contains a: 5, Lines contains Spark: 2
  ```

### Standalone 集群模式

Spark 自带的集群模式为 Standalone，一般用于开发调试。

- 修改配置文件

  配置集群 worker 节点，编辑 slaves 文件，输入以下内容：

  ```bash
  master
  slave1
  slave2
  ```

  编辑 spark-env.sh 文件，加入以下配置：

  ```bash
  JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
  SCALA_HOME=/usr/local/scala-2.11.12
  # 指定master主机名
  SPARK_MASTER_HOST=master
  # 指定master端口
  SPARK_MASTER_PORT=7077
  # 指定worker核数
  SPARK_WORKER_CORES=1
  # worker分配给执行器的内存
  SPARK_WORKER_MEMORY=1g
  # spark 配置文件安装路径
  SPARK_CONF_DIR=/usr/local/spark-2.4.7-bin-hadoop2.7/conf
  ```

- 启动集群模式

  在 /usr/local/spark-2.4.7-bin-hadoop2.7 路径下执行以下命令：

  ```bash
   sbin/start-all.sh 
  ```

- 功能验证

  执行以下命令通过spark-submit 来验证集群功能，spark-shell 也同理：

  ```bash
  bin/spark-submit --master spark://master:7077 /root/spark-demo.jar file:///root/test.txt
  ```

### 使用 yarn 集群模式

- 修改配置文件

  编辑 spark-env.sh，加入以下配置：

  ```bash
  JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
  SCALA_HOME=/usr/local/scala-2.11.12
  # spark 配置文件安装路径
  SPARK_CONF_DIR=/usr/local/spark-2.4.7-bin-hadoop2.7/conf
  # 指定 hadoop 配置文件路径
  HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
  ```

- 启动 hdfs、yarn服务，重启 spark 服务。

- 功能验证

  spark-shell 方式：

  进入 /usr/local/spark-2.4.7-bin-hadoop2.7 目录，启动 spark-shell：

  ```bash
  bin/spark-shell --master yarn --deploy-mode client
  ```

  启动后如果出现类似如下提示:

  ```bash
  20/12/13 11:58:41 ERROR cluster.YarnClientSchedulerBackend: YARN application has exited unexpectedly with state FAILED! Check the YARN application logs for more details.
  20/12/13 11:58:41 ERROR cluster.YarnClientSchedulerBackend: Diagnostics message: Application application_1607860172549_0002 failed 2 times due to AM Container for appattempt_1607860172549_0002_000002 exited with  exitCode: -103
  Failing this attempt.Diagnostics: [2020-12-13 11:58:40.183]Container [pid=1132,containerID=container_1607860172549_0002_02_000001] is running beyond virtual memory limits. Current usage: 289.8 MB of 1 GB physical memory used; 2.1 GB of 2.1 GB virtual memory used. Killing container.
  ...
  ```

  说明机器内存使用超出限制，可以把 yarn 对内存的相关检测关闭，在 yarn-site.xml 加上以下内容：

  ```xml
  <!--关闭虚拟内存检测-->
  <property>
      <name>yarn.nodemanager.vmem-check-enabled</name>
      <value>false</value>
  </property>
  <!--关闭物理内存检测-->
  <property>
      <name>yarn.nodemanager.pmem-check-enabled</name>
      <value>false</value>
  </property>
  ```

   重启 yarn 及 spark 服务即可。

  在 spark-shell 执行对应命令及结果，如下：

  ```bash
  scala> val textFile = spark.read.textFile("file:///root/test.txt")
  textFile: org.apache.spark.sql.Dataset[String] = [value: string]
  
  scala> textFile.count
  res0: Long = 5   
  ```

  spark-submit 方式：

  首先往 hdfs 上传一份文件，进入 /usr/local/hadoop/ 目录中，执行以下命令：

  ```bash
  bin/hadoop fs -put /root/test.txt /input/test.txt
  ```

  对 spark-demo.scala 代码中 spark.master 初始化值调整如下：

  ```scala
  val spark = SparkSession.builder().appName("app").config("spark.master", "yarn")
        .getOrCreate()
  ```

  重新打成 jar 包，并命名成 spark-yarn-demo.jar。

  进入 /usr/local/spark-2.4.7-bin-hadoop2.7 目录，执行以下命令：

  ```bash
  bin/spark-submit --class SparkDemo --master yarn --deploy-mode  cluster /root/spark-yarn-demo.jar /input/test.txt
  ```

  