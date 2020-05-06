---
title: Redis-HyperLogLog
description: 通常情况下，可以使用集合(set)进行唯一计数。集合所需要的内存，与存在它里面的元素数量成正比。在有大量元素的情况下，这个内存消耗是很大的。如果仅仅是为了统计不同元素的个数，又不需要很精确，Redis 提供的 HyperLogLog 数据结构就能够以很低的代价(最高 12KB 空间、标准误差 0.81%)满足需求。
categories: 
 - code
tags:
 - redis
---

------

## 前言

​	通常情况下，可以使用集合(**set**)进行唯一计数。集合所需要的内存，与存在它里面的元素数量成正比。在有大量元素的情况下，这个内存消耗是很大的。如果仅仅是为了统计不同元素的个数，又不需要很精确，**Redis** 提供的 **HyperLogLog** 数据结构就能够以很低的代价(最高 **12KB** 空间、标准误差 **0.81%**)满足需求。

## 存储类型

​	**HyperLogLog** 看起来像是一个新的数据结构，实际上，它的底层是 **String** 类型，采用两种方式存储对象。

- **稀疏(Sparse)**：当 **HLL** 对象大小小于 `hll-sparse-max-bytes(默认 3000)`时，采用稀疏矩阵存储。稀疏矩阵存储效率更高，但会消耗更多 **CPU** 资源。
- **稠密(Dense)**：当对象大小达到阈值，会转成稠密矩阵，这时才占用 **12KB** 空间。

## 相关命令

### [PFADD](<https://redis.io/commands/pfadd>)

添加一个或多个元素到指定 **key** 的 **HyperLogLog** 中，内部会更新，以反映到目前为止添加的唯一计数。

```shell
$ PFADD key element [element ...]
redis> pfadd hll_1 a
1
redis> pfadd hll_1 b
1
redis> pfadd hll_1 c
1
redis> pfadd hll_1 d e f
1
redis> pfadd hll_2 a b c D E F
1
```

### [PFCOUNT](<https://redis.io/commands/pfcount>)

获取指定 **keys** 的唯一计数。

```shell
$ PFCOUNT key [key ...]
redis> pfcount hll_1
6
redis> pfcount hll_2
6
redis> pfcount hll
9
```

### [PFMERGE](<https://redis.io/commands/pfmerge>)

将多个 **sourcekey** 合并到 **destkey** 中，**destkey** 会结合多个源，算出对应的唯一计数。

```shell
$ PFMERGE destkey sourcekey [sourcekey ...]
redis> pfmerge hll hll_1 hll_2
OK
```

## 参考

​	\[1\] [Redis 官方文档](<https://redis.io/topics/data-types-intro>)

​	\[2\] [钱文品. Redis 深度历险：核心原理与应用实践. 应用 4：四两拨千斤——HyperLogLog](<https://book.douban.com/subject/30386804/>)
