---
title: "HashMap相关知识点总结"
date: 2020-07-15T15:52:00+08:00
tags: ["Java基础", "HashMap"]
draft: true
---

> 基于 1.8.0_181 JDK 版本。

## HashMap、HashTable、ConcurrentHashMap区别

### HashMap 与 HashTable 区别

#### 线程安全

HashTable方法是线程安全的（使用了`synchronized`），HashMap是非线程安全。

#### 继承关系

HashTable继承自`Dictionary`，HashMap继承自`AbstractMap`，二者都实现了`Map`接口。

#### 是否允许`null`值

HashTable `key`，`value`都不允许`null`值，如果为`null`,则抛出空指针异常；HashMap `key` 可为 `null`值，但是这样的键只有一个，允许一个`key`或多个`key`的`value`为`null`。

#### 默认初始容量和扩容机制

HashTable 哈希数组初始容量为11，扩容方式为 old*2+1（源代码为`int newCapacity = (oldCapacity << 1) + 1`）;

HashMap 哈希数组初始容量为16，而且一定为2的指数，扩容方式为 old*2；

二者扩容的时机都为 实际`KV`大小 大于等于 临界值时。

#### 哈希值使用

HashTable使用对象的哈希值，HashMap重新计算哈希值。

#### 遍历方式

遍历方式内部都使用了`Iterator`，除此之外，HashTable还使用了`Enumeration`的方式，使用`Iterator`方式支持`fast-fail`,使用`Enumeration`则不支持。

> 什么是`fast-fail`，即代码在执行时会先进行某些校验，如果校验不通过，直接抛异常，避免后续代码执行出现更严重的异常。
>
> 在Java中，`fast-fail`从狭义上讲是针对多线程情况下的迭代器而言，具体可参看`ConcurrentModificationException`类上的注释。

### HashMap 与 ConcurrentHashMap 区别

二者内部实现方式内部都用了桶数组，但是`ConcurrentHashMap`对桶数组进行了分段加锁，而`HashMap`没有加锁操作，故前者是线程安全的，后者线程不安全。

## Java 8中Map中红黑树相关原理

### 什么是红黑树

## HashMap容量相关参数

## HashMap中hash方法原理

## 不同版本JDK中HashMap实现的区别以及原因

---

参考：

[HashMap、HashTable、ConcurrentHashMap区别](https://hollischuang.github.io/toBeTopJavaer/#/basics/java-basic/HashMap-HashTable-ConcurrentHashMap)

[重温数据结构：树及Java实现](https://blog.csdn.net/u011240877/article/details/53193877)

[重温数据结构：深入理解红黑树](https://blog.csdn.net/u011240877/article/details/53329023)

[HashMap 在 JDK 1.8 后新增的红黑树结构](https://blog.csdn.net/wushiwude/article/details/75331926)

[HashMap的容量、扩容](https://hollischuang.github.io/toBeTopJavaer/#/basics/java-basic/hashmap-capacity)

[HashMap中hash方法的原理](https://hollischuang.github.io/toBeTopJavaer/#/basics/java-basic/hash-in-hashmap)

[为什么HashMap的默认容量设置成16](https://hollischuang.github.io/toBeTopJavaer/#/basics/java-basic/hashmap-default-capacity)

[为什么HashMap的默认负载因子设置成 0.75 ](https://hollischuang.github.io/toBeTopJavaer/#/basics/java-basic/hashmap-default-loadfactor)

[HashMap实现原理分析](https://blog.csdn.net/vking_wang/article/details/14166593)