---
title: Java集合之LinkedHashMap
date: 2019-10-29
categories:
 - java
 - 集合
tags:
 - java
 - 集合
---

> 简要目标：学习Java集合中的LinkedHashMap，了解基本实现思想

### 1. 简说

&emsp;&emsp;LinkedHashMap继承自HashMap，可以理解它拥有HashMap的主要特性-高速的查询性能，此外，LinkedHashMap具备顺序性，通过accessOrder属性控制（插入顺序|访问顺序）,这给需要保持访问顺序性且高效查询的应用场景带来便利，HashMap可看[HashMap基本原理](./HashMap浅析.md)了解一波，那么本文就主要就两种集合类型的区别进行简要整理。

### 2. 顺序访问的秘诀
&emsp;&emsp;该特性的秘诀就在于LinkedHashMap内部维护了一个记录操作顺序的双向链表，在基于桶-链表结构的桶元素的基础上增加了两个引用属性，来保持对当前对象的后对象的快速定位：

```java
// HashMap的桶元素类
class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}
// LinkedHashMap的桶元素类
class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```
&emsp;&emsp;我觉得，可以用下面一个元素插入顺序说明该特性的核心功用：
假使： hash = x % 10，依次顺序插入13， 5， 15， 7， 19，55（数据随意，假的为了画图好看！），创建的双向链表如绿色的双向链接轨迹

![LinkedHashMap插入数据图](/images/191029-java_LinkedHashMap_1.png)

### 3. 基本操作

#### 3.1 查-get
&emsp;&emsp;查询查找是重新HashMap的get方法，但获取元素过程是使用的HashMap的getNode原生操作，只是后面多了一个根据accessOrder属性判断是否要对元素Link链表进行重排序（accessOrder为true的情况），说白了，如果accessOrder为true，就将当前元素链接到链表的最后去（调用afterNodeAccess方法）。
	
#### 3.2 删-remove
&emsp;&emsp;remove方法同样基本沿用HashMap的remove操作，在删除结束后，也要做一个操作，就是将当前元素的从链表中移除（调用afterNodeRemoval方法）。

#### 3.3 增、改
&emsp;&emsp; 

### 4. 扩容

### 5. 应用

### 6. 总结


