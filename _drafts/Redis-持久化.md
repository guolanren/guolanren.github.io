---
title: Redis-持久化
description: 
categories: 
 - code
tags:
 - redis
typora-root-url: ..

---

------

## 前言

​	两种持久化机制：RDB、AOF

## RDB

RDB 某一个时间点的快照，备份和灾难恢复。

默认开启

### 配置

save 900 1 (900 秒内有超过 1 个键发生改变，会进行一次 RDB 快照)

save 300 10

save 60 10000

stop-writes-on-bgsave-error yes BGSAVE 失败时，Redis 停止接受写入操作

rdbcompression yes 是否压缩

rdbchecksum yes: 快照文件末尾创建一个 CRC64 校验和。保存和加载快照额外消耗性能，选  no 降低数据损坏的抵抗力

### 命令

save 使用主线程同步转储，阻塞

bgsave 使用后台子进程转储，非阻塞 临时存储 temp-{bgsave-pid}.rdb 文件，完成后重命名 dbfilename 覆盖旧文件

## AOF

AOF 写入操作日志，服务器启动时重放

## 信息

​	info persistence

## 参考

​	\[1\] [Josiah L. Carlson 著. 黄健宏 译. Redis实战. 4.2 复制](<https://book.douban.com/subject/26612779/>)

​	\[2\] [黄鹏程,王左非 著. 梅隆魁 译. Redis 4.x Cookbook中文版. 第五章 复制](<https://book.douban.com/subject/30227261/>)

