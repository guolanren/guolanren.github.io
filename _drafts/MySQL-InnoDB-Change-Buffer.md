---
title: MySQL-InnoDB Change Buffer
description: 假设一张 innodb 表，维护了额外的非唯一的 Secondary Index（二级索引）。那么，在对这张表执行了 INSERT、UPDATE、DELETE 写操作（DML）时，如果相关联的页不在 Buffer Pool，数据库服务会从磁盘读取这些页，加载到 Buffer Pool 中，然后对其进行相应修改。最后通过一些刷盘时机，将这些修改过的脏页刷到磁盘中。至此，已经完成了数据的更新...
categories: 
 - code
tags:
 - mysql
 - innodb
---

------

## 前言

​	假设一张 **innodb** 表，维护了额外的非唯一的`Secondary Index（二级索引）`。那么，在对这张表执行了 **INSERT**、**UPDATE**、**DELETE** 写操作（**DML**）时，如果相关联的页不在`Buffer Pool`，数据库服务会从磁盘读取这些页，加载到`Buffer Pool`中，然后对其进行相应修改。最后通过一些刷盘时机，将这些修改过的脏页刷到磁盘中。至此，已经完成了数据的更新。

​	对于受影响的二级索引，需要同步维护。拿 **INSERT** 来说，一般聚簇索引以自增 **id** 为主键，可以实现顺序插入。但是对于普通的二级索引，如果有新的数据要插入，那么插入的位置基本都是随机的。每次更新，都伴随着至少一次的随机写磁盘，对于多写的系统来说，性能会受到很大的影响。

​	`Change Buffer（写缓冲）`能够缓冲写操作，稍后等待其他读操作，将对应的页加载到`Buffer Pool`后，再进行合并（**merge**），完成页的更新。

## 原理

​	对非唯一二级索引的修改操作，首先写到`Change Buffer`中。

## 结构

## 要求

- 二级索引（secondary index）
- 非唯一（not unique）

## 配置

## 参考

