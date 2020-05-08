---
title: 数据结构-Bloom-Filter
description: 布隆过滤器(Bloom Filter) 是 1970 年由布隆提出的，一种概率型(有一定的误报率)的数据结构。它能以极小的代价(CPU、内存)，判断集合中，某样东西一定不存在或者可能存在。
categories: 
 - code
tags:
 - 数据结构
---

------

## 前言

​	布隆过滤器(**Bloom Filter**) 是 **1970** 年由布隆提出的，一种概率型(有一定的误报率)的数据结构。它能以极小的代价(**CPU**、内存)，判断集合中，某样东西一定不存在或者可能存在。

## 应用场景

- 网页爬虫URL的去重，避免重复爬相同的网页
- 对视频、文章、博客等消息的历史推荐过滤，避免重复推荐
- 垃圾邮件过滤
- [缓存穿透](https://found.guolanren.online/code/2020/05/06/Redis-缓存问题与解决)

## 为什么 Bloom Filter

​	"判断一个东西是否存在"，仅根据需求，数组、列表、**HashSet**、**HashMap** 等数据结构能够解决。

- 数组/列表

  将已有的元素，添加到**数组**中，判断一个 **元素**是否存在于**数组**中，只需要遍历数组，对每一个元素做匹配。

  缺点：

  - 当集合有大量元素时，判断需要花费的时间很大
  - 仅仅是判断元素是否存在，花费的内存与元素成正比，当数据量大的时候，内存消耗自然高

- **HashSet/HashMap**

  通过哈希算法，添加元素与查找元素的时间复杂度都是 **O(1)**。

  缺点：

  - 跟数组一样，在大数据量的情况下，需要花费大量内存

​	如果仅仅是为了判断一个元素是否存在于集合，而且能够接受一定的错误率(可自己指定错误率)，**Bloom Filter** 就可以以低 **CPU**、内存的代价，满足需求。

## 实现原理

​	**Bloom Filter** 的底层，实际上是一个 **bit** 向量，也可以理解为一个 **bit** 数组。

![Bloom-Filter](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/Bloom-Filter.png?raw=true)

​	每当有元素要添加到集合中，都需要使用多个不同的哈希函数生成多个哈希值，对每个哈希值指向的 **bit** 置 **1**。如元素 **guolanren**

![guolanren-hash](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/guolanren-hash.png?raw=true)

​	**Bloom Filter** 的长度有限，而且哈希函数是会发生哈希冲突的，不同的元素存在相同哈希值的可能。

![guoyaozhan-hash](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/guoyaozhan-hash.png?raw=true)

​	判断元素是否存在于集合，只需要使用相同的哈希函数映射对 **bit** 位，检查每个位是否都是 **1**。如果存在不为 **1** 的位，就可以确定，这个元素不存在与集合。比如，**zhangsan** 哈希的结果，有不为 **1** 的值。

![zhangsan-hash](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/zhangsan-hash.png?raw=true)

​	之前有提到 **Bloom Filter** 是一个概率型数据结构，有一定的概率误判。比如，在拥有 **guolanren**、**guoyaozhan** 的集合，判断 **lisi** 是否存在。

![lisi-hash](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/lisi-hash.png?raw=true)

​	虽然哈希的结果，都是 **1**，但其实 **lisi** 并不存在于集合。

## 过滤器长度、哈希函数个数的选择

![Bloom-Filter-Param-Compare](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/Bloom-Filter-Param-Compare.png?raw=true)
> k：哈希函数个数，n：添加元素的个数，m：过滤器长度，p：误报率

​	由上图可发现：

- 在元素填满过滤器时，误报率近乎为 **1**，一般过滤器长度要考虑设置的足够大。

- 在过滤器长度足够大，其他条件相同的情况下，哈希函数的个数越多，误报率越低。实际上，哈希函数多会导致过滤器效率低，**bit** 位填满的速度更快。

​	参考的博客中，提供了公式，供过滤器长度、哈希函数的个数选择参考。

​	首先，过滤器的误判率有个预测公式：

![False-Positive-Probability](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/False-Positive-Probability.png?raw=true)

​	当 **n** 接近 **m**，或者大于等于 **m**，过滤器每次查询基本都会返回 **true**。所以，过滤器长度要设计的比预期添加元素个数大得多，以提高误报率。一般 **Bloom Filter** 的实现都提供误报率的设置，用户可以设置期望的误报率，并结合业务，根据以下公式，计算出过滤器合理的长度：

![Bloom-Filter-Length](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/Bloom-Filter-Length.png?raw=true)

​	至于合理的哈希函数个数，则可参考以下公式计算出：

![Hash-Amount](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/Hash-Amount.png?raw=true)

## 常用实现

### 		[Guava](<https://github.com/google/guava/wiki/HashingExplained#bloomfilter>)

​	**Guava** 中提供布隆过滤器的实现 `BloomFilter`，仅需提供 **Funnel** 就可以使用了。通过 `create(Funnel funnel, int expectedInsertions, double falsePositiveProbability)`获取 `BloomFilter`，默认的误报率为 **3%**。

```java
BloomFilter<Person> friends = BloomFilter.create(personFunnel, 500, 0.01);
for (Person friend : friendsList) {
  friends.put(friend);
}
// much later
if (friends.mightContain(dude)) {
  // the probability that dude reached this place if he isn't a friend is 1%
  // we might, for example, start asynchronously loading things for dude while we do a more expensive exact check
}
```

### 		[RedisBloom](<https://redisbloom.io/>)

​	**RedisBloom** 是 **Redis** 的一个概率数据类型模块，共提供了四种数据类型：**Bloom Filter**、**Cuckoo Filter**、**Count-Min-Sketch**、**Top-K**。其中，**Bloom** 和 **Cuckoo** 过滤器是用来推断(有一定的错误率)集合中是否存在某个 **item**。而 **Count-Min-Sketch** 以线性的内存空间估算出 **item** 数，**Top-K** 维护一个包含 **K** 个最频繁 **item** 的列表。详情请参考[官方文档](<https://redisbloom.io/>)或[Redis-RedisBloom](<https://found.guolanren.online/code/2020/05/06/Redis-RedisBloom/>)

## 删除支持

​	传统的 **Bloom Filter** 不支持删除操作，而 [RedisBloom](<https://found.guolanren.online/code/2020/05/06/Redis-RedisBloom/>) 中的 **Cuckoo Filter** 支持删除。

## 参考

​	\[1\] [Yound Chen. 详解布隆过滤器的原理，使用场景和注意事项](<https://zhuanlan.zhihu.com/p/43263751>)

​	\[2\] [semlinker. 布隆过滤器你值得拥有的开发利器](<https://segmentfault.com/a/1190000021136424>)

​	\[3\] [Abishek Bhat. Use the bloom filter, Luke!](<https://www.semantics3.com/blog/use-the-bloom-filter-luke-b59fd0839fc4/>)