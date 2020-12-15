---
date: 2020-12-14
title: "Spark基础知识总结"
tags: ["大数据", "Spark"]
---

## RDD

### 什么是RDD

RDD（Resilient Distributed Dataset），弹性分布式数据集，是Spark中最基本的计算逻辑抽象，它代表一个不可变、可分区、里面元素可并行计算的集合。

- 何谓弹性？

  - 数据可以存在磁盘中，也可以存在内存中；
  - 数据分布也是弹性的：
    - 这里弹性不是指可以动态扩展，而是容错机制。
    - RDD 会在多个节点上存储，就和 HDFS 分布式存储一样。 HDFS 文件切分成多个 Block 存储在多个节点中，而 RDD 是切分成多个 Partition，不同的 Partition 可能是在不同的节点上。
    - Spark 在读取 HDFS 的场景下，Spark 把 HDFS 的 Block 读到内存中就会抽象为 Partition。
    - Spark 计算结束，如果数据持久化到 HDFS 中，RDD 的每个 Partion就会存成一个文件，如果文件小于 128M，就可以理解成一个 Partion 对应一个Block，反之，就会切分成多个 Block，这样就是一个 Partion对应多个 Block。

- 分布式

  RDD 的数据是分布式存储的，也可以做分布式的计算。

- 数据集

  简单理解就是数据的集合。

> 因为 RDD 是分布式存储的，并且数据不可变，这也决定了它可以做并行计算。

## 算子

### 转换算子（ transformation）

Value 类型

双 Value 类型

K-V 算子

### 行动算子（actrion）