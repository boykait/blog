---
title: java多线程之AQS
date: 2019-11-22
categories:
  - java多线程
tags:
  - java
  - 多线程
---

### 1. 什么是AQS
&emsp;&emsp;前面Lock锁的常规使用部分介绍到了两种比较常用的锁ReentrantLock和ReentrantReadWriteLock，这两种锁实现尤其是后者给多线程的并发访问场景的带来了不小的性能提升，我们都知道，针对同一块临界资源，在使用锁的lock()方法锁住的时候，同一时刻只能有一个线程访问，知道锁的释放unlock()，还有两种锁实现都具备公平和非公平的能力，够强吧，这一切底层都归功于AQS(AbstractQueuedSynchronizer，抽象队列同步器)，除了Lock锁，还有其它的如:Semaphore都是依赖于AQS, 此外，我们还可以借助AQS模板功能实现自己的同步器。