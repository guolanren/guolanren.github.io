---
title: JVM-Class Data Sharing
description: 最初的 Class data sharing (CDS) 是 J2SE 5.0 新增的特性，主要是为了降低应用的启动时间，同时也能够减少内存的使用...
categories: 
 - code
tags:
 - jvm
 - jep
---

------

## 前言

​	最初的 **Class data sharing (CDS)** 是 **J2SE 5.0** 新增的特性，主要是为了降低应用的启动时间，同时也能够减少内存的使用。它通过加载系统的 **jar** 包到一个私有的内部表示，并转储为一个叫做`shared archive`的内存映射文件。开启该特性的 **jvm** 进程，会使用 **BootStrapClassLoader** 去加载`shared archive`文件中的类。多个 **jvm** 进程共享`shared archive`中的类的元数据，省去内存的额外消耗，以及免去解析这些固有的类所耗的时间。

​	它仅支持 **Java HotSpot Client VM**，垃圾收集器也仅支持 **serial garbage collector**。由于这些局限，早期并没有被广泛使用。在后续的更新迭代中，对 **CDS** 的一系列改进，才使得该特性慢慢受关注。

## JDK 5 (基础版本)

​	在启用 **CDS** 之前，需要先创建`shared archive`文件。

```shell
java -Xshare:dump
```

​	执行转储命令后，可以看到一些细节日志。

## JDK 6 (支持 server 模式)

## JDK 10 (AppCDS)

## 参考

​	\[1\] [](<https://book.douban.com/subject/34907497/>)

