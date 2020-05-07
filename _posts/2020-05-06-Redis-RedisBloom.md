---
title: Redis-RedisBloom
description: RedisBloom 是 Redis 的一个概率数据类型模块，共提供了四种数据类型：Bloom Filter、Cuckoo Filter、Count-Min-Sketch 、Top-K。其中，Bloom 和 Cuckoo 过滤器是用来推断(有一定的错误率)集合中是否存在某个 item。而 Count-Min-Sketch 以线性的内存空间估算出 item 数，Top-K 维护一个包含 K 个最频繁 itme 的列表。
categories:
 - code
tags:
 - redis
---

------

## 前言

​	**RedisBloom** 是 **Redis** 的一个概率数据类型模块，共提供了四种数据类型：**Bloom Filter**、**Cuckoo Filter**、**Count-Min-Sketch**、**Top-K**。其中，**Bloom** 和 **Cuckoo** 过滤器是用来推断(有一定的错误率)集合中是否存在某个 **item**。而 **Count-Min-Sketch** 以线性的内存空间估算出 **item** 数，**Top-K** 维护一个包含 **K** 个最频繁 **item** 的列表。

## 使用

```shell
# 下载 RedisBloom
$ mkdir /etc/redis-bloom
$ cd /etc/redis-bloom
$ git clone https://github.com/RedisBloom/RedisBloom.git
# 构建 RedisBloom
$ cd /etc/redis-bloom/RedisBloom
$ make
# 启动 Redis 服务时加载模块
$ redis-server --loadmodule /etc/redis-bloom/RedisBloom/redisbloom.so
# Redis 服务运行时加载模块
127.0.0.1:6379> MODULE LOAD /etc/redis-bloom/RedisBloom/redisbloom.so
# Redis 配置文件加载模块
# loadmodule /etc/redis-bloom/RedisBloom/redisbloom.so
# 初始 Bloom Filter 的错误率、初始大小
$ redis-server --loadmodule /etc/redis-bloom/RedisBloom/redisbloom.so INITIAL_SIZE 400 ERROR_RATE 0.004
```

## JRedisBloom (Java 客户端)

```xml
<dependency>
    <groupId>com.redislabs</groupId>
    <artifactId>jrebloom</artifactId>
    <version>1.2.0</version>
</dependency>
```

## Bloom Filter

### BF.RESERVE

```shell
# 结合错误率和初始化大小参数，创建一个空的布隆过滤器。
BF.RESERVE {key} {error_rate} {capacity} [EXPANSION expansion] [NONSCALING]
```

### BF.ADD

```shell
# 向布隆过滤器添加一个 item，如果过滤器不存在则自动创建
BF.ADD {key} {item}
```

### BF.MADD

```shell
# 向布隆过滤器添加一个或多个 item，如果过滤器不存在则自动创建
BF.MADD {key} {item} [item...]
```

### BF.INSERT

```shell
# 向布隆过滤器添加一个或多个 item，如果过滤器不存在则自动创建
# CAPACITY: 过滤器已存在则忽略此参数，不存在则使用该参数设置初始化大小
# ERROR: 过滤器已存在则忽略此参数，不存在则使用该参数设置错误率
# NOCREATE: 如果过滤器不存在，不会自动创建，且返回错误 
BF.INSERT {key} [CAPACITY {cap}] [ERROR {error}] [EXPANSION expansion] [NOCREATE]
[NONSCALING] ITEMS {item...}
```

### BF.EXISTS

```shell
# 判断 item 是否存在于过滤器中
BF.EXISTS {key} {item}
```

### BF.MEXISTS

```shell
# 判断 items 是否存在于过滤器中
BF.MEXISTS {key} {item} [item...]
```

### BF.INFO

```shell
# 返回 key 的详细信息
BF.INFO {key}
```

## Cuckoo Filter

### CF.RESERVE 

```shell
# 创建一个空的布谷鸟过滤器。它的错误率固定在 3% 左右，具体取决于过滤器容量的使用情况
CF.RESERVE {key} {capacity} [BUCKETSIZE bucketSize] [MAXITERATIONS maxIterations] [EXPANSION expansion]
```

### CF.ADD

```shell
# 向布谷鸟过滤器添加一个 item，如果过滤器不存在则自动创建
# 可多次添加相同的 item，并认为每个插入都是独立的
# 可以使用 CF.ADDNX 仅添加不存在的 item
CF.ADD {key} {item}
```

### CF.ADDNX

```shell
# 如果该 item 不存在，则添加到过滤器中
CF.ADDNX {key} {item}
```

### CF.INSERT、CF.INSERTNX

```shell
# 向过滤器添加一个或多个 item，如果过滤器不存在则自动创建
# CAPACITY: 过滤器已存在则忽略此参数，不存在则使用该参数设置初始化大小
# NOCREATE: 如果过滤器不存在，不会自动创建，且返回错误 
CF.INSERT {key} [CAPACITY {cap}] [NOCREATE] ITEMS {item ...}
CF.INSERTNX {key} [CAPACITY {cap}] [NOCREATE] ITEMS {item ...}
```

### CF.EXISTS

```shell
# 判断 items 是否存在于过滤器中
CF.EXISTS {key} {item}
```

### CF.DEL

```shell
# 从过滤器中移除该 item 一次，如果该 item 被添加多次，则仍然存在
# 删除不存在的 item，可能会导致删除其他的 item
CF.DEL {key} {item}
```

### CF.COUNT

```shell
# 返回该 item 在过滤器中的数量（估计值），如果仅仅想知道是否存在，使用 CF.EXISTS 更好
CF.COUNT {key} {item}
```

### CF.INFO

```shell
# 返回 key 的详细信息
CF.INFO {key}
```

## Count-Min-Sketch

​	后续有使用场景时再更新...

## Top-K

​		后续有使用场景时再更新...

## Bloom vs. Cuckoo

​	**Bloom Filter** 通常在插入 **item** 时，表现出更好的性能和伸缩性；而 **Cuckoo Filter** 在检查操作时更快，并且还支持删除 **item** 操作。

## 参考

​	\[1\] [RedisBloom 官方文档](<https://oss.redislabs.com/redisbloom/>)

