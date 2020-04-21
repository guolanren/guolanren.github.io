---
title: JDK源码-HashMap
description: 
categories: 
 - code
tags:
 - JDK源码
 - Map
---

------

## 前言

​	在分

## 数据结构

​	**HashMap** 底层使用一个 **Node** 数组存储数据，当某个桶存储的 **Node** 链表大小大于 **TREEIFY_THRESHOLD(8)** ，且整个 **Map** 储存元素的总大小大于 **MIN_TREEIFY_CAPACITY(64)** 时，该桶的 **Node** 会进一步转为 **TreeNode**，改用红黑树存储。而当

```java
/**
 * Basic hash bin node, used for most entries.  (See below for
 * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
 */
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
	...
}

/**
 * Entry for Tree bins. Extends LinkedHashMap.Entry (which in turn
 * extends Node) so can be used as extension of either regular or
 * linked node.
 */
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    ...
}

/**
 * The table, initialized on first use, and resized as
 * necessary. When allocated, length is always a power of two.
 * (We also tolerate length zero in some operations to allow
 * bootstrapping mechanics that are currently not needed.)
 */
transient Node<K,V>[] table;
```

## 初始化

​	初始化时，仅仅是对[大小](#大小)和[负载因子](#负载因子)进行设置，底层数组并不分配空间。只有在第一次插入元素时，才会对数组进行初始化 [resize()](#resize())。

```java
/**
 * Constructs an empty <tt>HashMap</tt> with the specified initial
 * capacity and load factor.
 *
 * @param  initialCapacity the initial capacity
 * @param  loadFactor      the load factor
 * @throws IllegalArgumentException if the initial capacity is negative
 *         or the load factor is nonpositive
 */
public HashMap(int initialCapacity, float loadFactor) {
    // 如果初始化大小小于 0，抛出非法参数异常
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    // HashMap 的最大大小为 MAXIMUM_CAPACITY（1 << 30）
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    
    // 负载因子不为负
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    // 计算出实际大小（大于等于 initialCapacity，且最小的 2 的幂的数）
    this.threshold = tableSizeFor(initialCapacity);
}

/**
 * Constructs an empty <tt>HashMap</tt> with the specified initial
 * capacity and the default load factor (0.75).
 *
 * @param  initialCapacity the initial capacity.
 * @throws IllegalArgumentException if the initial capacity is negative.
 */
public HashMap(int initialCapacity) {
    // 指定大小，默认负载因子
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

/**
 * Constructs an empty <tt>HashMap</tt> with the default initial capacity
 * (16) and the default load factor (0.75).
 */
public HashMap() {
    // 默认大小，默认负载因子
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

/**
 * Constructs a new <tt>HashMap</tt> with the same mappings as the
 * specified <tt>Map</tt>.  The <tt>HashMap</tt> is created with
 * default load factor (0.75) and an initial capacity sufficient to
 * hold the mappings in the specified <tt>Map</tt>.
 *
 * @param   m the map whose mappings are to be placed in this map
 * @throws  NullPointerException if the specified map is null
 */
public HashMap(Map<? extends K, ? extends V> m) {
    // 默认负载因子
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    // 将源 map 中的元素 put 进来
    putMapEntries(m, false);
}
```

## 关键属性

### DEFAULT_INITIAL_CAPACITY

默认的初始化大小 **16**，必须是 **2** 的幂。

- 为什么必须是 **2** 的幂？

  **HashMap** 在存储时，会使用 [hash(Object key)](#hash(Object key)) 根据 **Key** 的 **hashcode** 计算出最终的哈希值。最后使用 **i = (n - 1) & hash** 计算出索引值。**(n - 1)** 的结果正好相当于一个**低位掩码**，在进行 **&(与)** 时，可以让索引尽可能地分布均匀。

```java
/**
 * The default initial capacity - MUST be a power of two.
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```

## 关键方法

### hash(Object key)

扰动函数，高 **16** 位与低 **16** 位 **^(异或)**，计算 **key** 对应的哈希值。

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

- 为什么不直接使用 **key** 的 **hashcode** ？

  首先，**map** 存储的元素一般不会大于 **2<sup>16</sup>(65536)**。在使用 **i = (n - 1) & hash** 计算索引时你的 **hashcode** 设计不合理的情况下，一旦低位出现大量一致，就会哈希冲突。使用高位特征与低位特征，计算出来的结果尽可能地让每一位的变化都能够对最终结果产生影响。

- 为什么不用 **&** 或 **|** 而用 **^** ？

  **&** 容易让结果偏向 **0**；**|** 容易让结果偏向 **1**

- 为什么 **i = (n - 1) & hash** 要 **n - 1**？

  **map** 的容量都是 **2** 的幂，这种情况下的二进制形式，举 **2<sup>4</sup>(16)**为例：

  ```
  0000 0000 0000 0000 0000 0000 0001 0000
  ```

  与这样的结果进行 **&(与)** ，得到的结果大机率是 **0**（即使前面的哈希值已结合高低位特征，尽可能的散列）。

### putVal()

添加元素，返回被覆盖的值

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 如果底层数组为空，使用 resize() 初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 如果桶位为空，直接插入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

 