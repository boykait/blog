---
title: HashMap浅析
date: 2018-09-19 13:13:47
categories:
 - 后端
tags:
 - Java
---

### 1. 概述

&emsp;&emsp;1965年，Intel创始人之一的戈登·摩尔提出了经典的摩尔定律：当价格不变时，集成电路上可容纳的元器件的数目，约每隔18-24个月便会增加一倍，性能也将提升一倍。这个定律持续了超过了半个世界，虽然现在提升速率可能下降到三年翻一番。换而言之，随着硬件存储性能的提升，我们能够做更多的以空间换取时间的事情。从100000个员工的集合中以O(1)的时间复杂度查到某员工号对应的信息，用HashMap；方法传递的参数和类型不确定时，用HashMap；当前前提得保证线程安全性。  
&emsp;&emsp;本小文主要介绍HashMap初始化和扩容工作。


### 2. hash基本概念
#### 2.1. 哈希算法
&emsp;&emsp;接受任意长度的二进制输入值，对输入值做换算（hash），最终给出固定长度的二进制输出值；Hash算法不是某个固定的算法，它代表的是一类算法，具体换算可能各不相同。

#### 2.2. 哈希表
&emsp;&emsp;即散列表，一种数据结构，根据关键码值(Key value)而直接进行访问。给定表M，存在函数f(key)，对任意给定的关键字值key，代入函数后若能得到包含该关键字的记录在表中的地址，则称表M为哈希(Hash）表，函数f(key)为哈希(Hash) 函数

#### 2.3. 哈希碰撞
&emsp;&emsp;根据上面对哈希表的定义，有一种情况是，对不同的关键字可能得到同一散列地址，即k1≠k2，但f(k1)=f(k2)，这种情况就是哈希碰撞。如何解决哈希碰撞呢
拉链法（hashmap的解决方式）： 
将键转换为数组的索引(0-M-1)，但是对于两个或者多个键具有相同索引值的情况，将大小为M 的数组的每一个元素指向一个条链表，链表中的每一个节点都存储散列值为该索引的键值对，这就是拉链法

### 3. 初始化
&emsp;&emsp;HashMap类内部结构中定义的几个比较重要的变量：

```java
//默认初始化容量16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4
//最大容量2^30
static final int MAXIMUM_CAPACITY = 1 << 30;
//加载因子，默认为0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f;
//桶上对应的链表长度转换为树的阈值，即当链表长度超过8时，将链表转换为红黑树进行存储
static final int TREEIFY_THRESHOLD = 8;
//在resize后，将树转换为列表的阈值
static final int UNTREEIFY_THRESHOLD = 6;
// 桶中结构转化为红黑树对应的数组的最小大小，如果当前容量小于它，就不会将链表转化为红黑树，而是用resize()代替
static final int MIN_TREEIFY_CAPACITY = 64;
// 存储元素的数组，总是2的幂
transient Node<k,v>[] table; 
// 存放具体元素的集
transient Set<map.entry<k,v>> entrySet;
// 存放元素的个数，注意这个不等于数组的长度。
transient int size;
// 每次扩容和更改map结构的计数器
 transient int modCount;   
// 临界值 当实际节点个数超过临界值(容量*填充因子)时，会进行扩容
int threshold;
//实际填充因子
final float loadFactor;
```
&emsp;&emsp;HashMap的初始化方法有多种，默认初始化方法基本不做什么操作，只是将当前的加载因子设置为默认值：
```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
}
```
&emsp;&emsp;很多时候，我们需要根据程序具体的场景设计不同的初始化容量和加载因子：
```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
    
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

```
&emsp;&emsp;HashMap的初始化容量默认为1>>4即16，自定义初始化容量建议为2的幂，为什么需要这么定义？其实方便后面的位运算操作，在此，调用tableSizeFor方法找到大于初始化容量initialCapacity的最小的2幂长度作为阈值;首先，为什么要对cap做减1操作。int n = cap - 1; 
这是为了防止，cap已经是2的幂。如果cap已经是2的幂， 又没有执行这个减1操作，则执行完后面的几条无符号右移操作之后，返回的capacity将是这个cap的2倍，后面几个操作的目的是将为了将当前二进制元素后面的所有0都变成1，结果再+1就可以得到大于初始化容量initialCapacity最小的2的幂数了。

### Put方法

HashMap提供的方法最重要的无非存值put和取值get,先看一下put方法的实现：  
1000 1111
```java
 public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // 初次存数据resize HashMap对象
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // 利用(n - 1) & hash 定位到新元素应存放的tab下标，并调用newNode创建节点（如果节点存在，则仅更新节点数据）  
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            //位于tab数组对应拉链的第一个数据？直接赋值
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //是否为TreeNode 红黑树节点类型，是则按红黑树节点操作插入节点
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //拉链长度大约最大拉链阈值？是则将拉链转换为树结构
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
            //如果之前已经存放了对应的key？则替换新值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                //将旧的值作为整个put方法的返回值进行返回
                return oldValue;
            }
        }
        ++modCount;
        //添加完节点后，load factor*current capacity超过了当前的阈值，进行resize操作
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }    
```
&emsp;&emsp;在put方法调用过程中，有几个点需要重点进行分析
- hash(key)计算key对应的hash值
- resize所完成的工作
- 红黑树方式插入元素
- treeifyBin将拉链转换为红黑树

#### hash及哈希桶索引定位
```java
//计算hash码
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
//定位下标
  (n - 1) & hash
````
&emsp;&emsp;hash(key)方法差不多就一句代码，获取元素Key对应的HashCode码，然后用高16bit位和低16bit位进行异或，具体作用是是什么呢？主要是从速度、功效、质量来考虑的，这么做可以在bucket的n比较小的时候，也能保证考虑到高低bit都参与到hash的计算中设计者权衡了speed， utility 和 quality， 将高16位与低16位异或来减少这种影响。设计者考虑到现在的hashCode分布的已经很不错了，而且当发生较大碰撞时也用树形存储降低了冲突。仅仅异或一下，既减少了系统的开销，也不会造成的因为高位没有参与下标的计算(table长度比较小时)，从而引起的碰撞。将高16bit与低16bit异或之后，hash的分布变的复杂一些，更“接近”随机，但仍然是均匀的。估计作者是从实际使用的角度出发，因为一般情况下，key的分布也符合“局部性原理”，低比特位相同的概率大于异或后仍然相同的概率，从而降低了碰撞的概率。  
&emsp;&emsp;当计算完元素对应的hash码后，通过(n - 1) &  hash定位bucket桶下标，对于任意给定的对象，只要它的hashCode()返回值相同，那么计算得到的hash值总是相同的。我们首先想到的就是把hash值对table长度取模运算，这样一来，元素的分布相对来说是比较均匀的。
但是模运算消耗还是比较大的，我们知道计算机比较快的运算为位运算，因此JDK团队对取模运算进行了优化，使用上面代码2的位与运算来代替模运算。这个方法非常巧妙，它通过 “(table.length -1) & h” 来得到该对象的索引位置，这个优化是基于以下公式：x mod 2^n = x & (2^n - 1)。我们知道HashMap底层数组的长度总是2的n次方，并且取模运算为“h mod table.length”，对应上面的公式，可以得到该运算等同于“h & (table.length - 1)”。这是HashMap在速度上的优化，因为&比%具有更高的效率。在JDK1.8的实现中，还优化了高位运算的算法，将hashCode的高16位与hashCode进行异或运算，主要是为了在table的length较小的时候，让高位也参与运算，并且不会有太大的开销。   
####resize
&emsp;&emsp;resize方法的作用主要是初始化table或进行扩容（扩容成之前的两倍）：
```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        //设置初始化容量和阈值、或设置容量和阈值为当前的两倍
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    //没有next直接放到新的bucket中
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    //
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        //存放与之前bucket相同下标的元素列表
                        Node<K,V> loHead = null, loTail = null;
                        //存放与新bucket下标的元素列表
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            //e.hash & oldCap比较确定元素是否需移动
                            
                            //不需要，，串成链表
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            //需要，串成链表
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        
                        //分别将两个新链表存入对应的bucket桶中
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
&emsp;&emsp;注意上面的e.hash & oldCap的原理如下：
```
(e.hash & oldCap) 得到的是 元素的在数组中的位置是否需要移动,示例如下

示例1：
   e.hash=10 0000 1010
   oldCap=16 0001 0000
     &   =10 0000 0000       比较高位的第一位 0
   结论：元素位置在扩容后数组中的位置没有发生改变

示例2：
   e.hash=17 0001 0001
   oldCap=16 0001 0000
     &   =1  0001 0000      比较高位的第一位   1
结论：元素位置在扩容后数组中的位置发生了改变，
新的下标位置是原下标位置+原数组长度

(e.hash & (oldCap-1))得到的是下标位置,示例如下

        e.hash=10 0000 1010
      oldCap-1=15 0000 1111
           &  =10 0000 1010

     e.hash=17 0001 0001
   oldCap-1=15 0000 1111
        &  =1  0000 0001

新下标位置
    e.hash=17 0001 0001
  newCap-1=31 0001 1111    newCap=32
       &  =17 0001 0001    1+oldCap = 1+16

元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit，
因此新的index就会发生这样的变化：
0000 0001->0001 0001
```

&emsp;&emsp;上面如果对应的节点类型为TreeNode，调用split方法重新进行划分， 原理同上面计算普通链表是否需要分链类似，先根据e.hash & bit判断元素是否需要更新bucket下标，由此得到两个新的链表，然后根据新链表的长度决定树化或者逆树化操作。
```java
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
            TreeNode<K,V> b = this;
            // Relink into lo and hi lists, preserving order
            TreeNode<K,V> loHead = null, loTail = null;
            TreeNode<K,V> hiHead = null, hiTail = null;
            int lc = 0, hc = 0;
            for (TreeNode<K,V> e = b, next; e != null; e = next) {
                next = (TreeNode<K,V>)e.next;
                e.next = null;
                if ((e.hash & bit) == 0) {
                    if ((e.prev = loTail) == null)
                        loHead = e;
                    else
                        loTail.next = e;
                    loTail = e;
                    ++lc;
                }
                else {
                    if ((e.prev = hiTail) == null)
                        hiHead = e;
                    else
                        hiTail.next = e;
                    hiTail = e;
                    ++hc;
                }
            }

            if (loHead != null) {
                if (lc <= UNTREEIFY_THRESHOLD)
                    tab[index] = loHead.untreeify(map);
                else {
                    tab[index] = loHead;
                    if (hiHead != null) // (else is already treeified)
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {
                if (hc <= UNTREEIFY_THRESHOLD)
                    tab[index + bit] = hiHead.untreeify(map);
                else {
                    tab[index + bit] = hiHead;
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
        }
```

### get方法
&emsp;&emsp;get方法根据key获取对应的值，这个方法是HashMap对外体现的一个最重要的功能，实现O(1)时间复杂度内获取数据就靠它了，匹配值时先计算对应的hash值然后再equals方法比较：
```java
public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
}
// java8允许设置默认值
return (e = getNode(hash(key), key)) == null ? defaultValue : e.value;

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 第一个就是要找的元素，找到并返回
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            //通过红黑树结构寻找元素
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            //循环普通bucket链表找到对应的元素
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}

```

待学学红黑树相关的再回来继续写。使用红黑树结构还可以避免哈希碰撞攻击[link](https://yq.aliyun.com/articles/92194?t=t1)





