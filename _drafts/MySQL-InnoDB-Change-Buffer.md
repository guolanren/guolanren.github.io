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

​	对于受影响的二级索引，需要同步维护。拿 **INSERT** 来说，一般聚簇索引以自增 **id** 为主键，可以实现顺序插入。但是对于普通的二级索引，如果有新的数据要插入，

