---
title: java多线程之synchronized
date: 2019-11-17
categories:
  - java
tags:
  - java
  - 多线程
---

> 学习synchronized的常规使用之间的区别以及其实现机制

#### 1. 什么是synchronized
&emsp;&emsp;在多线程并发访问的情况下，存在可见性、原子性以及顺序性等问题，如果不能很好的控制，就可能得到不可预期的结果，synchronized就是用于对并发访问的控制的一种锁。可以通过为方法、方法块加上synchronized，就能够保持多个线程对该段程序进行互斥访问。下面给出一组synchronized的常规操作：

```java
/**
 * 说明：主要测试线程在执行synchronized修饰的代码时，
 * 其他线程对哪些进行并发访问不产生阻塞。
 */


public class SynchronizedTest2 {

    void method() {
        System.out.println("执行普通方法");
    }

    synchronized void syncMethod() {
        System.out.println("执行第一个synchronized同步方法开始");
        try {
            Thread.sleep(15000);
            System.out.println("执行第一个synchronized同步方法");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("执行第一个synchronized同步方法结束");
    }

    synchronized void syncMethod1() {
        System.out.println("执行第二个synchronized同步方法");
    }

    void syncMethod2() {
        System.out.println("准备进入synchronized同步方法块");
        synchronized (this) {
            System.out.println("synchronized执行同步方法块");
        }

    }
    static synchronized void syncMethod3() {
        System.out.println("执行静态synchronized同步方法");
    }

     void syncMethod4() {
        System.out.println("准备进入类对象锁同步方法块");
        synchronized (SynchronizedTest.class) {
            System.out.println("执行类对象锁的同步方法块");
        }

    }

    public static void main(String[] args) throws Exception {
        SynchronizedTest2 st = new SynchronizedTest2();
        new Thread(() -> st.syncMethod()).start();
        Thread.sleep(4000); // 延迟几秒，主要保证第一个线程先于后面的线程执行
        new Thread(() -> st.method()).start();
        new Thread(() -> st.syncMethod1()).start();
        new Thread(() -> st.syncMethod2()).start();

        new Thread(() -> st.syncMethod3()).start();
        new Thread(() -> st.syncMethod4()).start();
    }
}

// 输出结果：    
执行第一个synchronized同步方法开始    
执行普通方法    
准备进入同步方法块    
执行静态synchronized同步方法        
准备进入类对象锁同步方法块        
执行类对象锁的同步方法块    
执行第一个synchronized同步方法    
执行第一个synchronized同步方法结束    
执行同步方法块    
执行第二个synchronized同步方法    

```    

&emsp;&emsp;说明：当第一个线程在执行synchronized修饰方法期间，有如下表现：
- 可执行该对象的普通方法；
- 可执行synchronized修饰静态方法；
- 可执行锁类对象修饰的代码块；
- 不可执行该对象其他synchronized修饰方法；
- 不可执行锁该对象修饰的代码块。

#### 2. synchronized锁原理
&emsp;&emsp;上面以一个简单的demo呈现了synchronized的作用，这一切的缘由就在于对象锁。


接下来我们就需要分析为什么synchronized能够利用对象锁做到这样的保障。    
&emsp;&emsp;利用javap对上例demo的字节码文件进行反编译，抽取目标部分：    
```java
 void method();
    descriptor: ()V
    flags:
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2               
         3: ldc           #3              
         5: invokevirtual #4             
         8: return
		 
  synchronized void syncMethod();
    descriptor: ()V
    flags: ACC_SYNCHRONIZED
    Code:
      ...
    
  synchronized void syncMethod1();
    descriptor: ()V
    flags: ACC_SYNCHRONIZED
    Code:
     ...

  void syncMethod2();
    descriptor: ()V
    flags:
    Code:
      stack=2, locals=3, args_size=1
         0: getstatic     #2                 
         3: ldc           #14               
         5: invokevirtual #4               
         8: aload_0
         9: dup
        10: astore_1
        11: monitorenter
        12: getstatic     #2             
        15: ldc           #15              
        17: invokevirtual #4               
        20: aload_1
        21: monitorexit
        22: goto          30
        25: astore_2
        26: aload_1
        27: monitorexit
        28: aload_2
        29: athrow
        30: return

  static synchronized void syncMethod3();
    descriptor: ()V
    flags: ACC_STATIC, ACC_SYNCHRONIZED
    ...
	
  void syncMethod4();
    descriptor: ()V
    flags:
    Code:
      stack=2, locals=3, args_size=1
         0: getstatic     #2                 
         3: ldc           #17                
         5: invokevirtual #4                 
         8: ldc           #18                
        10: dup
        11: astore_1
        12: monitorenter
        13: getstatic     #2                 
        16: ldc           #19                 
        18: invokevirtual #4                 
        21: aload_1
        22: monitorexit
        23: goto          31
        26: astore_2
        27: aload_1
        28: monitorexit
        29: aload_2
        30: athrow
        31: return  
```

&emsp;&emsp;可以看到，被synchronized修饰的方法块多了两条指令：monitorenter、monitorexit，被synchronized修饰的方法的flags中会多一个ACC_SYNCHRONIZED标志，依次了解一波。

##### monitorenter & monitorexit
&emsp;&emsp;


