---
title: JVM内存模型
date: 2019-09-20
categories:
 - JVM
 - java
tags:
 - JVM
 - java
---

> 分析对比Java模型，由此开启学习JVM的路程，

### 1. JVM内存模型
&emsp;&emsp;基本概念也不再copy了，直接上自己画的内存模型图：
JVM内存模型图
![Java7-JVM内存模型图](/images/190920-jvm_memory_model_1.png)

- java7之前的方法区（永久代）中主要存放：类信息、常量、静态变量、即时编译器编译后的代码，
- java7存储在永久代的部分数据就已经转移到Java Heap或者Native memory。但永久代仍存在于JDK 1.7中，并没有完全移除，譬如符号引用(Symbols)转移到了native memory；字符串常量池(interned strings)和类的静态变量(class statics)转移到了Java heap。
- java8中，取消永久代，类信息、编译后的代码数据放于元空间(Metaspace)，元空间使用的是本地内存。	

为什么要对方法区做这样的处理呢？主要是为了：
```java
1、字符串存在永久代中，容易出现性能问题和内存溢出。
2、永久代大小不容易确定，PermSize指定太小容易造成永久代OOM
3、永久代会为 GC 带来不必要的复杂度，并且回收效率偏低。
4、Oracle 可能会将HotSpot 与 JRockit 合二为一，实现统一
```
&emsp;&emsp;下一篇了解研究一下JVM的另外一重点垃圾回收问题。
