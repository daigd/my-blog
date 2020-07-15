---
title: "Mysql架构与历史"
date: 2020-07-13T17:55:22+08:00
tags: ["MySql", "MySql架构与历史"]
draft: false
---

## MySql逻辑架构

下图展示了`MySql`的逻辑架构：

![MySql逻辑架构](/mysql/img/MySql服务器逻辑架构.png)

客户端层：该层并不为`MySql`独有，基于网络的客户端/服务器的工具或服务都有类似构架，诸如连接处理、授权认证、安全等服务均在这层处理。

服务层：