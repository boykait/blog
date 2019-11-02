---
title: Java并发之CoucurrentHashMap
date: 2019-10-29
categories:
 - java
 - 集合
 - 并发
tags:
 - java
 - 并发
 - 集合
---

> 简要目标：HashMap是开发过程中较为常用的数据结构之一，但在并发访问的情况下会出现线程不安全性，由此诞生的ConcurrentHashMap学习Java并发中的CoucurrentHashMap，本文主要了解基本实现思想，当然本文主要就JDK1.8版本的实现进行简要分析记录，并比较JDK1.7和1.8版本的实现区别

### 1. 大纲

```
1. JDK 1.7版本实现思想
2. JDK 1.8版本实现
```
### 3. ConcurrentHashMap-1.8
&emsp;&emsp;在JDK1.8版本中，摒弃了1.7中的Segument数组来保证线程并发访问的安全特性，而是利用了部分代码加锁synchronized+CAS操作来保证对象的写操作或扩容等操作，具体存结构和1.8版本的HashMap一样，会在适当时机进行数据链表/红黑树的转换操作。看看其内部几个核心属性，都是使用了使用volatile来保证数据更改对多个线程的可见性访问：

```java
// ConcurrentHashMap类
transient volatile Node<K,V>[] table;
private transient volatile Node<K,V>[] nextTable;
private transient volatile int sizeCtl;

// Node
 static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;
}


```

#### 3.1 初始化

#### 3.2 put
&emsp;&emsp;put是戏最多的一个操作，涉及到表项初始化、扩容、链表/红黑树转换等操作，所以先来重点分析一波。**有个疑问未解，fh<0时按TreeBin处理？**    
```java
public V put(K key, V value) {
        return putVal(key, value, false);
    }

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 计算对象hash值
    int hash = spread(key.hashCode());
    int binCount = 0;
    // put核心过程，自旋以保证数据被正确put
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 桶数组初始化，这个阶段需要保证只有一个线程进行初始化操作
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 如果桶数组对应下标为null，通过原生CAS方法将数据存在该下标位置上
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 如果当前桶内的所有对象正在进行迁移，加入进去help，一起迁
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            // synchronized 保证并发put
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
						// 遍历 
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
							// 存在则修改
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
							// 不存在则new
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                // 树化操作
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

- initTable过程
&emsp;&emsp;初始化table表，看看内部是如何保证线程的安全性的?
```java
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
			// 当 sizeCtl 小于0时，表示有其他线程在进行初始化操作，便让出调用权
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            // 使用CAS保证设置sizeCtl=-1，让其他线程在上面的(sc = sizeCtl) < 0成立
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    // 初始化表
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
						// 如果table初始化完成，表示table的容量，默认是table大小的0.75倍，
						// 公式算0.75（n - (n >>> 2)）
                        sc = n - (n >>> 2);
                    }
                } finally {
 					
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```
&emsp;&emsp;








