---
date: 2020-12-15
title: "Java虚拟机常用参数"
tags: ["Java", "JVM"]
---

## 查看虚拟机使用的垃圾收集器

命令：java -XX:+PrintCommandLineFlags -version

```bash
java -XX:+PrintCommandLineFlags -version
```

- Java 8

  ```bash
  -XX:InitialHeapSize=266235904 -XX:MaxHeapSize=4259774464 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:-UseLargePagesIndividualAllocation -XX:+UseParallelGC
  java version "1.8.0_181"
  Java(TM) SE Runtime Environment (build 1.8.0_181-b13)
  Java HotSpot(TM) 64-Bit Server VM (build 25.181-b13, mixed mode)
  ```

- JDK-13

  ```bash
  -XX:G1ConcRefinementThreads=4 -XX:GCDrainStackTargetSize=64 -XX:InitialHeapSize=266235904 -XX:MaxHeapSize=4259774464 -XX:MinHeapSize=6815736 -XX:+PrintCommandLineFlags -XX:ReservedCodeCacheSize=251658240 -XX:+SegmentedCodeCache -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseG1GC -XX:-UseLargePagesIndividualAllocation
  openjdk version "13" 2019-09-17
  OpenJDK Runtime Environment (build 13+33)
  OpenJDK 64-Bit Server VM (build 13+33, mixed mode, sharing)
  ```

- JDK-15

  ```java
  -XX:ConcGCThreads=1 -XX:G1ConcRefinementThreads=4 -XX:GCDrainStackTargetSize=64 -XX:InitialHeapSize=266235904 -XX:MarkStackSize=4194304 -XX:MaxHeapSize=4259774464 -XX:MinHeapSize=6815736 -XX:+PrintCommandLineFlags -XX:ReservedCodeCacheSize=251658240 -XX:+SegmentedCodeCache -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseG1GC -XX:-UseLargePagesIndividualAllocation
  openjdk version "15.0.1" 2020-10-20
  OpenJDK Runtime Environment (build 15.0.1+9-18)
  OpenJDK 64-Bit Server VM (build 15.0.1+9-18, mixed mode, sharing)
  ```

  垃圾收集器相关参数说明：

  | 参数               | 描述                                                         |
  | ------------------ | ------------------------------------------------------------ |
  | UseConcMarkSweepGC | 使用ParNew + CMS + Serial Old 收集器组合进行内存回收，Serial Old 作为 CMS 收集器出现 Concurrent Mod Failure 失败后的备用收集器。 |
  | UseParallelGC      | 使用 Parallel Scavenge + Serial Old 收集器组合进行内存回收。 |
  | UseG1GC            | 使用 G1 （Garbage First）收集器进行内存回收。                |

  由此可知，Java 8 默认使用 Parallel Scavenge + Serial Old 垃圾收集器，JDK 13 和 JDK 15默认使用 G1 垃圾收集器。

## 





