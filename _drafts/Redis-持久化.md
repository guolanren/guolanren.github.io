---
title: Redis-持久化
description: 
categories: 
 - code
tags:
 - redis
---

------

## 前言

​	Redis 是一个基于内存的数据库，一旦 Redis 进程故障退出，所有数据将会丢失。Redis 通过两种持久化机制，提供了防止数据丢失的能力，这两种机制分别是 RDB、AOF。

## RDB

​	RDB 是 Redis 某一个时间点的数据快照，可用于数据备份和灾难恢复。默认存储在数据目录下的 `dump.rdb` 文件中。RDB 持久化默认开启。

### 配置

- save \<seconds\> \<changes\>

  表示距上一次快照时间的时间内，有超过多少个键发生改变，就会触发 BGSAVE，进行一次 RDB 快照。

  默认配置（可配置多个）：

  ```properties
  save 900 1
  save 300 10
  save 60 10000
  ```

- stop-writes-on-bgsave-error

  BGSAVE 失败时，Redis 停止接受写入操作。默认 yes。

- rdbcompression

  是否压缩 RDB 文件。默认 yes。

- rdbchecksum

  在快照文件末尾创建一个 CRC64 校验和。设置 yes，在保存和加载快照时会额外性能开销，设置 no，则会降低数据损坏的抵抗力。默认 yes。

- dbfilename

  RDB 文件名。默认 dump.rdb

### 命令

- SAVE

  使用主线程进行同步转储，阻塞。

- BGSAVE

  调用 fork 创建子进程转储，非阻塞。临时存储文件 `temp-{bgsave-pid}.rdb` 。完成转储后，重命名为 dbfilename 覆盖旧文件。

### 执行

​	除了直接调用上述命令 SAVE \| BGSAVE，Redis 服务器收到某些指令也会触发快照转储。

- SHUTDOWN

  当 Redis 服务器收到来自客户端的 SHUTDOWN 指令时，会先执行 SAVE，阻塞后续的请求。

- SYNC

  作为 master 的服务器，收到复制 SYNC 指令时，会先执行 BGSAVE。

## AOF

​	AOF（Append Only File），会在数据目录下生成 `appendonly.aof` 文件，以文件末尾追加的形式，记录了服务器收到的写入命令（RESP，Redis 序列化协议）。命令并不是直接写入磁盘的，Redis 维护了一个缓冲区，命令先往该缓冲区写，最后通过 Linux 调用 fsync() 同步到磁盘中（该过程是阻塞的），才真正的完成持久化。服务器启动时，可以通过重放 AOF 文件的命令来达到数据的恢复。默认是关闭的。

### 配置 

- appendonly

  是否开启 AOF 持久化。默认 no。

- appendfilename

  AOF 文件名。默认 appendonly.aof。

- appendfsync

  用于调整 fsync 的频率。有三个值供选择

  - always

    每有一条写命令，就调用 fsync 一次。能够保证进程意外退出时，只丢失一条命令。但 fsync 是阻塞的，并且受写磁盘性能的限制，频繁调用会导致 Redis 性能降低。

  - everysec（默认）

    每秒进行一个 fsync 调用。能够保证进程意外退出时，最多丢失一秒的数据。

  - no

    不调用 sync。由系统决定何时将数据从缓冲区写入到磁盘。大多 Linux 系统，这个频率是每 30 秒。

- auto-aof-rewrite-min-size

  表示文件大小小于该值，AOF 重写不会被触发。默认 64 mb。

- auto-aof-rewrite-percentage

  Redis 将记录最后一次 AOF 重写的文件大小，如果当前 AOF 文件大小增长超过了这个百分比，就会触发一次 AOF	重写。默认 100。

### 命令

- BGREWRITEAOF

  Redis 不断地将写命令追加到 AOF 文件中，写命令一旦很多，将导致 AOF 文件很大。在使用该 AOF 文件进行数据恢复时，需要花费很多时间。通过 AOF  重写 （AOF rewrite） 压缩 AOF 文件，可以减小文件大小。原理很简单，就是将多个写命令进行合并。如对一个键进行过多次设置操作，将只保留最后一条写操作命令。

  BGREWRITEAOF 命令，可以启动子进程进行重写操作。子进程由于写时复制机制的限制，不能够访问其创建后新产生的数据。所以，主进程会将写命令先写到 aof_rewrite_buf_blocks 缓冲区。当子进程完成重写后，主进程再将缓冲区的命令写入到新的 AOF 文件，覆盖旧 AOF 文件。

## RDB & AOF

### 比较

|                | RDB                                                      | AOF                                                          |
| -------------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| 数据一致性     | 进程意外退出，将丢失最近一次快照之后的所有数据（写操作） | 进程意外退出<br />always 丢失 1 条写命令<br />everysec 丢失 1 秒内的数据<br />no 一般丢失 30 秒左右的数据 |
| 文件大小       | 较小                                                     | 较大                                                         |
| 重启数据恢复   | 较快                                                     | 较慢                                                         |
| 持久化性能开销 | 持久化性能开销，随内存数据增大而增大                     | 持久化性能开销，随写请求频率增大而增大                       |
|                |                                                          |                                                              |

### 结合使用

​	配置参数 aof-use-rdb-preamble（默认 yes） ，是 Redis 4.x 开始提供的混合持久化功能。在重写 AOF 文件时，Redis 会把数据以 RDB 的格式作为 AOF 文件的开始部分写入。这之后，则继续使用 AOF 格式追加写命令。这种混合格式，通过 RDB 的压缩格式实现了数据的快速加载，也保留了 AOF 数据一致性的优势。

## 修复

​	如果操作系统奔溃，RDB \| AOF 文件可能会损坏，可以通过 Redis 提供的工具尝试进行修复。

- redis-check-rdb \<rdb-file-name\>
- redis-check-aof [--fix] \<file.aof\>

## 信息

​	INFO PERSISTENCE 可以查看当前的持久化信息。

## 参考

​	\[1\] [Josiah L. Carlson 著. 黄健宏 译. Redis实战. 4.1 持久化选项](<https://book.douban.com/subject/26612779/>)

​	\[2\] [黄鹏程,王左非 著. 梅隆魁 译. Redis 4.x Cookbook中文版. 第6章 持久化](<https://book.douban.com/subject/30227261/>)

