---
title: "Scala基础语法笔记"
date: 2020-12-03
tags: ["Scala基础"]
---

> 该笔记是笔者学习Mybatis源码时，用 Scala 写单元测试用例时，遇到的基础语法知识点扫盲笔记。

## 关键字

### trait

特性（特征）是一些字段和行为的集合，功能上要求是原子化的，即每个特性粒度要细，功能要单一（接口隔离原则）。

### with

通过 `with` 关键字，一个类可以扩展多个特性：

```scala
abstract class UnitSpec extends AnyFlatSpec with GivenWhenThen
  with Matchers with OptionValues with Inside with Inspectors with MockFactory
{
}
```

## Scala 变量

### 变量与常量

在 Scala 中， 用 `var` 声明变量，`val`声明常量。