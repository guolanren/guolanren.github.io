---
title: 数据结构-Bloom-Filter
description: 布隆过滤器(Bloom Filter) 是 1970 年由布隆提出的，一种概率型(有一定的误报率)的数据结构。它能以极小的代价(CPU、内存)，判断集合中，某个元素一定不存在或者可能存在。
categories: 
 - code
tags:
 - 数据结构
---

------

## 前言

​	布隆过滤器(**Bloom Filter**) 是 **1970** 年由布隆提出的，一种概率型(有一定的误报率)的数据结构。它能以极小的代价(**CPU**、内存)，判断集合中，某个元素一定不存在或者可能存在。

## 应用场景

- 网页爬虫 **URL** 的去重，避免重复爬相同的网页
- 对视频、文章、博客等消息的历史推荐过滤，避免重复推荐
- 垃圾邮件过滤
- [缓存穿透](https://found.guolanren.online/code/2020/05/06/Redis-缓存问题与解决)

## 为什么 Bloom Filter

​	"判断集合中，元素是否存在"，可以使用数组、列表、**HashSet**、**HashMap** 等数据结构能够解决。

- 数组/列表

  将已有的元素，添加到数组/列表中，判断一个元素是否存在于数组/列表中，只需要遍历数组/列表，对每一个元素做匹配。缺点是，当集合有大量元素时，判断需要花费的时间很大；花费的内存与元素成正比，当数据量大的时候，内存消耗自然高。

- **HashSet/HashMap**

  通过哈希算法，添加元素与查找元素的时间复杂度都是 **O(1)**。而缺点跟数组一样，在大数据量的情况下，需要花费大量内存。


​	如果仅仅是为了判断一个元素是否存在于集合，而且能够接受一定的错误率(可自己指定错误率)的话，**Bloom Filter** 就可以以低 **CPU**、内存的代价，满足需求。

## 实现原理

​	**Bloom Filter** 的底层，实际上是一个 **bit** 向量，也可以理解为一个 **bit** 数组。

![Bloom-Filter](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/Bloom-Filter.png?raw=true)

​	每当有元素要添加到集合中，都需要使用多个不同的哈希函数生成多个哈希值，对每个哈希值指向的 **bit** 置 **1**。如元素 **guolanren**

![guolanren-hash](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/guolanren-hash.png?raw=true)

​	**Bloom Filter** 的长度有限，不同的元素存在相同哈希值的可能。

![guoyaozhan-hash](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/guoyaozhan-hash.png?raw=true)

​	判断元素是否存在于集合，只需要使用相同的哈希函数，映射到 **bit** 位，检查每个位是否都是 **1**。如果存在不为 **1** 的位，就可以确定，这个元素不存在于集合。比如，**zhangsan** 哈希的结果，有不为 **1** 的值。

![zhangsan-hash](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/zhangsan-hash.png?raw=true)

​	之前有提到 **Bloom Filter** 是一个概率型数据结构，有一定的概率误判。比如，在仅有 **guolanren**、**guoyaozhan** 两个元素的集合中，判断 **lisi** 是否存在。虽然哈希的结果，都是 **1**，但其实 **lisi** 并不存在于集合。

![lisi-hash](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/lisi-hash.png?raw=true)

## 过滤器长度、哈希函数个数的选择

![Bloom-Filter-Param-Compare](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/Bloom-Filter-Param-Compare.png?raw=true)
> k：哈希函数个数，n：添加元素的个数，m：过滤器长度，p：误报率

​	由上图可发现：

- 在元素填满过滤器时，误报率近乎为 **1**，一般过滤器长度要考虑设置的足够大。

- 在过滤器长度足够大，其他条件相同的情况下，哈希函数的个数越多，误报率越低。实际上，哈希函数多会导致过滤器效率低，**bit** 位填满的速度更快。

​	参考的博客中，提供了公式，供过滤器长度、哈希函数的个数选择参考。

​	首先，过滤器的误判率有个预测公式：

![False-Positive-Probability](https://github.com/guolanren/gallery/blob/master/found/2020-05-07-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8/False-Positive-Probability.png?raw=true)

​	当 **n** 接近 **m**，或者大于等于 **m**，过滤器每次查询基本都会返回 **true**。所以，过滤器长度要设计的比预期添加元素个数大很多，以降低误报率。一般 **Bloom Filter** 的实现都提供误报率的设置，用户可以设置期望的误报率，并结合业务，根据以下公式，计算出过滤器合理的长度：

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

### [Redisson](<https://github.com/redisson/redisson/wiki/6.-%E5%88%86%E5%B8%83%E5%BC%8F%E5%AF%B9%E8%B1%A1#68-%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8bloom-filter>)

​	**Redisson** 利用 **Redis** 实现了 **Java** 分布式[布隆过滤器（Bloom Filter）](http://static.javadoc.io/org.redisson/redisson/3.10.0/org/redisson/api/RBloomFilter.html)。所含最大比特数量为`2^32`。

```java
RBloomFilter<SomeObject> bloomFilter = redisson.getBloomFilter("sample");
// 初始化布隆过滤器，预计统计元素数量为55000000，期望误差率为0.03
bloomFilter.tryInit(55000000L, 0.03);
bloomFilter.add(new SomeObject("field1Value", "field2Value"));
bloomFilter.add(new SomeObject("field5Value", "field8Value"));
bloomFilter.contains(new SomeObject("field1Value", "field8Value"));
```

### 		[RedisBloom](<https://redisbloom.io/>)

​	**RedisBloom** 是 **Redis** 的一个概率数据类型模块，共提供了四种数据类型：**Bloom Filter**、**Cuckoo Filter**、**Count-Min-Sketch**、**Top-K**。其中，**Bloom** 和 **Cuckoo** 过滤器是用来推断(有一定的错误率)集合中是否存在某个 **item**。而 **Count-Min-Sketch** 以线性的内存空间估算出 **item** 数，**Top-K** 维护一个包含 **K** 个最频繁 **item** 的列表。详情请参考[官方文档](<https://redisbloom.io/>)或 [**Redis-RedisBloom**](<https://found.guolanren.online/code/2020/05/06/Redis-RedisBloom/>)。

```java
public class RedisBloomBF {

    private Client client;

    @Before
    public void before() {
        client = new Client("127.0.0.1", 6379);
    }

    @Test
    public void testBloomFilter() {
        for (long i = 0; i < 100_000_000L; i++) {
            client.add("simpleBloom", Long.toString(i));
        }
        long count = 0;
        for (long i = 0; i < 2 * 100_000_000L; i++) {
            if (client.exists("simpleBloom", Long.toString(i))) {
                count++;
            }
        }
        System.out.println(count);
    }
}
```

## 粗略比较

- `Guava`：虽然批量添加接口，但由于使用本地内存，可以快速构建过滤器。在分布式应用中，多个应用需要各自维护过滤器。一般场景下，对过滤器的一致性要求不高。
- `Redisson`：基于 **Redis** 构建，没有提供批量添加的接口。如果需要构建大数据量的过滤器，需要很长时间。在分布式应用中可以共享过滤器。
- **RedisBloom**：基于 **Redis** 构建，**Redis** 需要加载该模块。提供了批量添加的接口，相比 **Redisson**，可快速构建过滤器，但远远慢于 **Guava**。在分布式应用中可以共享过滤器。

## 删除支持

​	传统的 **Bloom Filter** 不支持删除操作，而 [**RedisBloom**](<https://found.guolanren.online/code/2020/05/06/Redis-RedisBloom/>) 中的 **Cuckoo Filter** 支持删除。

## 参考

​	\[1\] [Yound Chen. 详解布隆过滤器的原理，使用场景和注意事项](<https://zhuanlan.zhihu.com/p/43263751>)

​	\[2\] [semlinker. 布隆过滤器你值得拥有的开发利器](<https://segmentfault.com/a/1190000021136424>)

​	\[3\] [Abishek Bhat. Use the bloom filter, Luke!](<https://www.semantics3.com/blog/use-the-bloom-filter-luke-b59fd0839fc4/>)