---
title: Redis-复制
description: redis 复制...
categories: 
 - code
tags:
 - redis
typora-root-url: ..
---

------

## 前言

​	Redis 单实例存在单点故障问题，而且为了缓解单实例的压力，生产环境往往都使用集群。Redis 集群的基础是复制，而 Redis 提供的复制能够使主实例（master）的数据复制到多个从实例（slave）中，从而获取容错及水平扩展的能力。

## 配置

```shell
# 通过 utils/install_server.sh 脚本创建一个新的 Redis 实例（salve）
shell> vi 6380.conf
	replicaof 127.0.0.1 6379
	# 如果 6379 实例设置密码，则需要配置认证
	masterauth ***
```

​	启动从实例后，可以分别从 6379、6380 实例中通过 **'info replication'** 查看复制信息。

```text
# ===============================================================
# 主实例
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=6380,state=online,offset=42,lag=1
master_replid:183a1f6ad7808740d2fb1ab463404512de42e698
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:42
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:42

# ===============================================================
# 从实例
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
master_last_io_seconds_ago:7
master_sync_in_progress:0
slave_repl_offset:56
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:183a1f6ad7808740d2fb1ab463404512de42e698
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:56
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:56
```

## 原理

重新同步机制

​	部分重新同步

​	完全重新同步：主实例 BGSAVE 转储 RDB 文件，将文件发给从实例，从实例清空内存所有数据，再讲RDB文件导入。

从实例第一次连接主实例，进行的是完全重新同步

主实例 根据 backlog 决定部分/完全



命令， slaveof  host port

slaveof no one





复制过程

1）从服务器：连接（重连）主服务器，发送 SYNC 命令

2）主服务器：接收 SYNC 命令，执行 BGSAVE，使用缓冲区 backlog 记录之后所执行的写命令

2）从服务器：根据配置来决定是继续使用先有的数据（如果有的话）来处理客户端的请求，还是向客户端返回错误

3）主服务器：BGSAVE 执行完，向从服务器发送 RDB 文件，发送期间继续使用缓冲区记录被写命令

3）从服务器：丢弃所有旧数据（如果有的话），开始载入主服务器发来的 RDB 文件

4）主服务器：RDB 文件发送完毕，开始向从服务器发送 backlog 中的写命令

4）从服务器：完成对 RDB 文件的载入，像往常异常接受命令请求

5）主服务器：backlog 中的写命令发送完毕之后，每执行的一次写命令，都向从服务器发送相同的写命令

5）从服务器：执行主服务器发来的 backlog 中的写命令。之后，接收并执行主服务器创来的每个写命令



### 主从链