---
title: JDK源码-HashMap
description: HashMap 源码解读...
categories: 
 - code
tags:
 - JDK源码
 - Map
---

------

## 数据结构

### 	Node

​	**map** 将键值对封装成 **Node** 进行存储，底层维护一个 **Node<K,V>[] table**。

### 	TreeNode

​	**Node** 的子类。当 **key** 不相等，但哈希冲突时，节点会以链表的形式，挂在数组上。

​	一旦链表通过添加节点，长度大于等于 [TREEIFY_THRESHOLD(8)](#TREEIFY_THRESHOLD) 时，在 [resize()](#resize()) 时会触发 [treeifyBin(Node<K,V>[] tab, int hash)](#treeifyBin(Node<K,V>[] tab, int hash))，链表可能会转成红黑树。

​	红黑树通过移除节点，长度小于等于 [UNTREEIFY_THRESHOLD(6)](#UNTREEIFY_THRESHOLD) 时，在 [resize()](#resize()) 时会触发 [untreeify(map)]()，红黑树会转成链表。

### 	table

​	**map** 的底层数据结构。

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

​	初始化时，仅仅是对大小和[负载因子](#DEFAULT_LOAD_FACTOR)进行设置，底层数组并不分配空间。只有在第一次插入元素时，才会对数组进行初始化 [resize()](#resize())。

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
    // 将源 map 中的键值对 put 进来
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

### DEFAULT_LOAD_FACTOR

默认负载因子 **0.75f**，数组元素

- 为什么需要负载因子？

  如果没有负载因子，数组应该在什么时候进行扩容？随着键值对不断增加，在不进行扩容的情况下，会频繁哈希冲突，一个桶位的节点越来越多，查找效率会变低。

  或者说，负载因子过于大，扩容不及时，也会引起上述问题；而如果负载因子过于小，频繁扩容导致，扩容性能消耗大，且内存利用不充分。

```java
/**
 * The load factor used when none specified in constructor.
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

### MIN_TREEIFY_CAPACITY

链转树最小容量 **64**，当触发 [treeifyBin(Node<K,V>[] tab, int hash)](#treeifyBin(Node<K,V>[] tab, int hash))，会先判断数组长度是否达到 **64**，再决定链转树还是扩容。

```java
/**
 * The smallest table capacity for which bins may be treeified.
 * (Otherwise the table is resized if too many nodes in a bin.)
 * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
 * between resizing and treeification thresholds.
 */
static final int MIN_TREEIFY_CAPACITY = 64;
```

### TREEIFY_THRESHOLD

链转树链长度的阈值 **8**。当链表长度大于等于 **8**，会触发 [treeifyBin(Node<K,V>[] tab, int hash)](#treeifyBin(Node<K,V>[] tab, int hash))。

```java
/**
 * The bin count threshold for using a tree rather than list for a
 * bin.  Bins are converted to trees when adding an element to a
 * bin with at least this many nodes. The value must be greater
 * than 2 and should be at least 8 to mesh with assumptions in
 * tree removal about conversion back to plain bins upon
 * shrinkage.
 */
static final int TREEIFY_THRESHOLD = 8;
```

### UNTREEIFY_THRESHOLD

树转链树大小的阈值 **6**。在 [split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit)](#split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit)) 时，当遇到树大小小于等于 **6**，会触发 [untreeify(HashMap<K,V> map)]()。

```java
/**
 * The bin count threshold for untreeifying a (split) bin during a
 * resize operation. Should be less than TREEIFY_THRESHOLD, and at
 * most 6 to mesh with shrinkage detection under removal.
 */
static final int UNTREEIFY_THRESHOLD = 6;
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

  首先，**map** 存储的键值对总数一般不会大于 **2<sup>16</sup>(65536)**。在使用 **i = (n - 1) & hash** 计算索引时你的 **hashcode** 设计不合理的情况下，一旦低位出现大量一致，就会哈希冲突。使用高位特征与低位特征，计算出来的结果尽可能地让每一位的变化都能够对最终结果产生影响。

- 为什么不用 **&** 或 **|** 而用 **^** ？

  **&** 容易让结果偏向 **0**；**|** 容易让结果偏向 **1**

- 为什么 **i = (n - 1) & hash** 要 **n - 1** ？

  **map** 的容量都是 **2** 的幂，这种情况下的二进制形式，举 **2<sup>4</sup>(16)**为例：

  ```
  0000 0000 0000 0000 0000 0000 0001 0000
  ```

  与这样的结果进行 **&(与)** ，得到的结果大机率是 **0**（即使前面的哈希值已结合高低位特征，尽可能的散列）。

### putVal()

添加键值对(key, value 均可为 null)，返回被覆盖的值。

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 如果底层数组为空，使用 resize() 初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        // 初始化数组长度
        n = (tab = resize()).length;
    // 临时变量存储对应索引位头结点，如果桶位为空，直接插入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 待插入的 key 是否等于头 Node 的 key
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            // 将当前节点设置为待修改节点
            e = p;
        // 如果是树节点
        else if (p instanceof TreeNode)
            // 待插入的 key/value，插入到红黑树中
            // 如果有 key 相等的 Node，返回该 Node
            // 否则，直接新建节点，返回 null
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 计算链表大小，后面判断是否需要转红黑树
            for (int binCount = 0; ; ++binCount) {
                // 链表遍历完，仍没有出现相同 key 的 Node
                if ((e = p.next) == null) {
                    // 尾插法，插入新节点到链表尾部
                    p.next = newNode(hash, key, value, null);
                    // 遍历第一个 Node 时，binCount 为 0，所以需要 -1
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        // 到了阈值 TREEIFY_THRESHOLD(8) 转红黑树
                        treeifyBin(tab, hash);
                    // 结束
                    break;
                }
                // 遍历过程中，遇到 key 相等的 Node，直接结束遍历
                // 此时 e 已经指向该 key 相等的 Node
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                // p 设置为下一个遍历节点
                p = e;
            }
        }
        // 如果不为空，e 就是待修改节点
        if (e != null) { // existing mapping for key
            // 获取旧 value
            V oldValue = e.value;
            // 根据 onlyIfAbsent 判断是否覆盖旧值
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            // 空方法，LinkedHashMap 才有具体实现
            afterNodeAccess(e);
            return oldValue;
        }
    }
    // modified count 加 1，用于迭代器(iterator)快速失败，抛 ConcurrentModificationException 异常
    ++modCount;
    // map 大小加 1，如果大于阈值
    if (++size > threshold)
        // 扩容
        resize();
    // 空方法，LinkedHashMap 才有具体实现
    afterNodeInsertion(evict);
    // 直接插入，没有覆盖，返回 null
    return null;
}
```

### resize()

初始化或双倍扩容数组。扩容之后，将原来 **Node** 插入新数组中(原索引位置或 **2** 的幂的偏移量索引位)。最后返回新数组。

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // 扩容
    if (oldCap > 0) {
        // 如果原数组已经大于 HashMap 定义的最大容量（1 << 30）
        if (oldCap >= MAXIMUM_CAPACITY) {
            // 将阈值设置为 Integer.MAX_VALUE
            threshold = Integer.MAX_VALUE;
            // 返回原数组
            return oldTab;
        }
        // 将新数组长度扩大 2 倍
        // 如果新数组长度小于 HashMap 定义的最大容量，且原数组长度大于等于默认的初始化大小 16
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 新的阈值扩大为原阈值的 2 倍
            newThr = oldThr << 1; // double threshold
    }
    // 初始化，如果构造 map 时指定了初始大小，阈值将根据初始大小，计算出实际大小（大于等于 initialCapacity，且最小的 2 的幂的数）
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // 初始化，如果没有指定初始化大小，使用默认大小
    else {               // zero initial threshold signifies using defaults
        // 默认初始化大小 16
        newCap = DEFAULT_INITIAL_CAPACITY;
        // 阈值为，默认初始化大小与默认负载因子的乘积，16 * 0.75 = 12
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 如果新阈值为 0
    if (newThr == 0) {
        // 计算阈值
        float ft = (float)newCap * loadFactor;
        // 如果新数组长度小于 HashMap 定义的最大容量（1 << 30），且计算出的阈值小于最大容量，新的阈值为计算阈值
        // 否则为 Integer.MAX_VALUE（直到 MAX_VALUE 都不会再扩容了）
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    // 设置 map 阈值属性
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 使用新长度申请新数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    // 设置 map 表属性
    table = newTab;
    // 如果原数组不为空，遍历数组，将 Node 插入到新数组
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            // Node 不为空
            if ((e = oldTab[j]) != null) {
                // 将原数组当前 Node 链表设置为空
                oldTab[j] = null;
                // 单独 Node 节点
                if (e.next == null)
                    // 当前 Node 移动到新数组的原索引位置或2的幂的偏移量索引位
                    newTab[e.hash & (newCap - 1)] = e;
                // 如果是树节点
                else if (e instanceof TreeNode)
                    // 进行树节点的插入
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // Node 链表，使用尾插法（保留原顺序）
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        // 使用临时变量，存储当前节点的 next 节点
                        next = e.next;
                        // oldCap 是原数组长度，用以区分树节点属于高位索引还是低位索引
                        if ((e.hash & oldCap) == 0) {
                            // 低位索引的第一个节点，设置为链表头节点
                            if (loTail == null)
                                loHead = e;
                            // 非头节点，设置低位链表的末尾节点的 next 为当前遍历节点
                            else
                                loTail.next = e;
                            // 设置当前遍历节点为链表末尾节点
                            loTail = e;
                        }
                        else {
                            // 高位索引的第一个节点，设置为链表头节点
                            if (hiTail == null)
                                hiHead = e;
                            // 设置当前遍历节点为链表末尾节点
                            else
                                hiTail.next = e;
                            // 设置当前遍历节点为链表末尾节点
                            hiTail = e;
                        }
                    // 遍历 Node 链表
                    } while ((e = next) != null);
                    if (loTail != null) {
                        // 低位链表插入到低位桶
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        // 高位链表插入到高位桶
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    // 返回新数组
    return newTab;
}
```

### split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit)

将红黑树分为高低位两棵树，插入到对应的桶

```java
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
    TreeNode<K,V> b = this;
    // Relink into lo and hi lists, preserving order
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    // 遍历红黑树节点
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        // 使用临时变量，存储当前节点的 next 节点
        next = (TreeNode<K,V>)e.next;
        // 将当前节点的 next 节点设置为空
        e.next = null;
        // bit 是原数组长度，用以区分树节点属于高位索引还是低位索引
        if ((e.hash & bit) == 0) {
            // 低位索引的第一个节点，设置为链表头节点
            if ((e.prev = loTail) == null)
                loHead = e;
            // 非头节点，设置低位链表的末尾节点的 next 为当前遍历节点
            else
                loTail.next = e;
            // 设置当前遍历节点为链表末尾节点
            loTail = e;
            // 计数器加 1
            ++lc;
        }
        else {
            // 高位索引的第一个节点，设置为链表头节点
            if ((e.prev = hiTail) == null)
                hiHead = e;
            // 非头节点，设置高位链表的末尾节点的 next 为当前遍历节点
            else
                hiTail.next = e;
            // 设置当前遍历节点为链表末尾节点
            hiTail = e;
            // 计数器加 1
            ++hc;
        }
    }

    if (loHead != null) {
        // 低位链表计数器是否小于"非树"阈值(6)
        if (lc <= UNTREEIFY_THRESHOLD)
            // 低位链表转成普通列表，插入低位桶
            tab[index] = loHead.untreeify(map);
        else {
            // 低位链表插入到低位桶
            tab[index] = loHead;
            // 如果为空，表示所有节点都到低位链表，已经是红黑树结构
            if (hiHead != null) // (else is already treeified)
                // 转成红黑树结构
                loHead.treeify(tab);
        }
    }
    if (hiHead != null) {
        // 高位链表计数器是否小于"非树"阈值(6)
        if (hc <= UNTREEIFY_THRESHOLD)
            // 高位链表转成普通列表，插入高位桶
            tab[index + bit] = hiHead.untreeify(map);
        else {
            // 高位链表插入到高位桶
            tab[index + bit] = hiHead;
            // 如果为空，表示所有节点都到高位链表，已经是红黑树结构
            if (loHead != null)
                // 转成红黑树结构
                hiHead.treeify(tab);
        }
    }
}
```

- 为什么要分高低位？

  节点插入到新的索引位的计算规则为 **i = (n - 1) & hash**，由于双倍扩容，**n** 是原来的 **2** 倍。

  举个例子(取低 **8** 位)：

  原数组长度 **16 (0001 0000) **-> 新数组长度 **32 (0010 0000)**

  原 **n - 1 (0000 1111)** -> 新 **n - 1 (0001 1111)**

  **key** 的 **hash** 是不变的，由此可见，新的索引位仅取决于多出的那一位(刚好对应原数组长度的二进制)。通过这个 **bit**，直接分出两个链表，插入到对应的索引位上(高低位索引相差的偏移量就是 **bit** 的大小)。

### treeifyBin(Node<K,V>[] tab, int hash)

对指定 **hash** 的链表转换成红黑树结构

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 如果数组长度小于 MIN_TREEIFY_CAPACITY(64)
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        // 直接扩容，让节点散列，最终是否需要转红黑树，由 resize() 方法决定
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            // 将普通 Node 转成 TreeNode
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                // 设置将首节点
                hd = p;
            else {
                // 设置遍历的当前节点的上一个节点
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        // 遍历链表，生成 hd 树节点链表
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            // 转红黑树
            hd.treeify(tab);
    }
}
```

