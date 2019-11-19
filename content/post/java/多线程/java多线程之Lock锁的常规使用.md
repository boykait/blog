---
title: java多线程之Lock锁的常规使用
date: 20199-11-19
categories:
  - java
tags:
  - java
  - 多线程
---

> 记录一些锁的基本使用方法和场景

### 什么Lock锁
&emsp;&emsp;Lock定义的是一个接口，主要包含获取锁、释放锁以及创建Condition的方法：    
```java
void lock(); // 直接上锁
void lockInterruptibly(); // 可中断上锁
boolean tryLock();   // 尝试上锁，无限等
boolean tryLock(long time, TimeUnit unit); // 尝试上锁，有限等
void unlock();  // 解锁
Condition newCondition();  // 可利用Condition实现线程间的等待通知
```

### 可重入锁和可重入读写锁
&emsp;&emsp;关于Lock的使用主要是使用可重入锁ReentrantLock和可重入读写锁ReentrantReadWriteLock，下面来做一些尝试。
#### ReentrantLock
&emsp;&emsp;ReentrantLock是实现了Lock接口，内部干事的主要是实现了一个静态同步器Sync(后面在研究AQS的时候再好好琢磨)，内部实现公平和非公平两种方式获取锁，可以通过new时进行指定`new ReentrantLock(true)`，演示最简单的用例，实现变量a自增，毫无压力，走：    
```java
public class LockTest {
    Lock lock = new ReentrantLock();
    int a = 0;

    public void add() {
        lock.lock();
        try {
            a++;
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        LockTest lt = new LockTest();
        for (int i = 0; i < 100; i++) {
            new Thread(() -> lt.add()).start();
        }

        // 等待打印结果
        Thread.sleep(2000);
        System.out.println("结果：" + lt.a);
    }
}
```       

#### ReentrantReadWriteLock
&emsp;&emsp;总有些场景就是想在读和写上做点文章，觉得读操作是无状态的，不会对我们的数据产生影响，大家都可以共享同时读，写的时候嘛，同一时刻就独占，只能一个写，注意有写操作的时候，就不许读了，嗯，符合常理，不然又读又写确实会乱套，因此，ReentrantReadWriteLock就产生了啊，可重入，读写分离，这波，我想装个BO，整个稍微复杂一点的，用它来实现一个简单的缓存，继续走：

```java
package com.boykait.locktest;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.locks.ReentrantReadWriteLock;

class Cache<K, V> {
    private Map<K, V> cacheMap;
    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public Cache() {
        this.cacheMap = new HashMap<>(16);
    }

    /**
     * 读
     *
     * @param key key
     * @return value
     */
    public V getFromCache(K key) {
        lock.readLock().lock();
        try {
            return cacheMap.get(key);
        } finally {
            lock.readLock().unlock();
        }
    }

    /**
     * 写
     *
     * @param key   key
     * @param value value
     */
    public void putToCache(K key, V value) {
        lock.writeLock().lock();
        try {
            cacheMap.put(key, value);
        } finally {
            lock.readLock().unlock();
        }
    }

    /**
     * 获取大小
     *
     * @return size
     */
    public int getSizeFromCache() {
        lock.readLock().lock();
        try {
            return cacheMap.size();
        } finally {
            lock.readLock().unlock();
        }
    }

    /**
     * 清空缓存
     */
    public void clearCache() {
        lock.writeLock().lock();
        try {
            cacheMap.clear();
        } finally {
            lock.readLock().unlock();
        }
    }
}

public class LockTest1 {
    public static void main(String[] args) throws InterruptedException {
        long start = System.currentTimeMillis();
        Cache<Integer, Integer> cache = new Cache<>();
        int size = 300000;
        List<Thread> list = new ArrayList<>(size);
        for (int i = 0; i < size; i++) {
            Thread t = new Thread(() -> cache.getFromCache(1));
            list.add(t);
        }
        list.forEach(Thread::start);
        list.forEach(t -> {
            try {
                t.join();
            } catch (InterruptedException e) {
            }
        });

        System.out.println((System.currentTimeMillis() - start));
    }
}
```

&emsp;&emsp;这个例子执行速度是27秒左右，测试的时候还做了一个ReentrantLock版本的(就是换一下ReentrantReadWriteLock)，速度在43秒左右，看来读得时候，共享读模式的ReentrantReadWriteLock还是有不少优势的哇。

### Condition用法

### 公平VS非公平

思考：Lock是如何实现可见性的？

