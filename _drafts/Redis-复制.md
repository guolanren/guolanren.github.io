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

