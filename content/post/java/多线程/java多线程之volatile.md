---
title: java多线程之volatile
date: 2019-11-15
categories:
  - 多线程
  - java
tags:
  - 多线程
  - java
--- 

#### 1. 讲讲什么是volatile？
&emsp;&emsp;在JMM模型中，每个线程有私有的工作内存(寄存器、缓存等)，多个线程之间主要通过共享内存空间进行通信，那么在多个线程对同一块主存数据的操作就会引发数据的不一致性问题，若线程1对主存的副本进行修改未及时刷新到主存中，可能会使得线程2读取到的就是脏数据，那么我们的修改操作也会随之丢失，所有，volatile关键字的主要作用就是来决绝多核CPU缓存数据的不一致性问题，主要体现在三个方面：    
- 1. 保证线程对数据的操作对其他线程具备可见性；
- 2. 保证有序性，防止编译器优化重排序引入的问题；
- 3. 需要注意的是，volatile不保证数据操作的原子性。

#### 2. 如何保证可见性？
&emsp;&emsp;首先，volatile的主要作用是变量，假如线程1和线程2均将变量a拷贝到各自的私有工作空间，当线程1对a进行重新赋值等操作，赋完值后，会立马将数据刷新到主存中去，线程2随后若要使用变量a，会发现当前的变量a已经失效了，需要重新从主存中进行读取，然后在进行相关的操作，这就保证了线程1对a的写操作对线程2是可见的。如下图：

![volatile多线程写和读](/images/191115_java_threads_volatile_1.png)

##### 2. 再深入说说？
&emsp;&emsp;保证可见性即需要保证缓存的一致性，主要有两种方式：锁总线和锁缓存，前者是CPU级别独占总线，性能无疑会下降，所以处理器主要就使用的锁缓存的技术，在翻译成机器指令的过程中，会对对被volatile修饰的变量的赋值操作进行特殊处理，即增加LOCK指令前缀，那么在执行这条指令时，相当于会连续执行如下原子指令，以上面变量a为例：
```
1. assagin: 为变量进行赋值操作,线程1的工作空间副本 a=1
2. store：将工作内存的对象传送到主存中
3. write： 将对象写入主存 主存a=1
```
&emsp;&emsp;此时，主存中的变量a就有了最新的数据1，但是线程2是如何得知数据发生了变更了呢？ 主要通过[缓存一致性协议](https://juejin.im/entry/5d67e77c5188253eff7d6e56)（MESI，Modified-Exclusive-shared-Invalid）来保证的，每个CPU会通过嗅探总线上的一些信号来更新自己缓存行的状态，如果探测到其它CPU进行了write写操作，更新自己的缓存行状态为Invalid无效。线程2在对相应的工作内存中操作副本变量a时，发现a数据是无效的，会强制从主存中进行读取:
```
1. read: 从主存中读取
2. load： 从主存中加载到工作内存
3. use：使用该副本变量
```
OK，以上就完成了可见性的保障！

#### 3. 谈谈有序性？
&emsp;&emsp;先谈谈JMM中的Happens-Before，在JMM中，在一个线程中，或不同的线程中。如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须存在Happens-Before关系。值得一提的是，我们说两个操作存在Happens-Before关系。这里指的操作的结果。并非执行顺序。
    
 - 1、程序次序： 在一个线程内，一段代码的执行结果是有序的。依然会有指令重排，但是不论怎么重排序，结果都是按照代码顺序生成的不会变。
 - 2、监视器锁： 无论在单线程环境还是多线程环境，对于同一个锁来说，一个线程对这个锁解锁之后，另一个线程获取了这个锁，则它能看到前一个线程的操作结果。
 - 3、volatile变量： 如果一个线程先去写一个volatile变量，然后一个线程去读这个变量，那么这个写操作的结果一定对读的这个线程可见，前面介绍的缓存一致性就保证了这个事情。
 - 4、线程启动： 在主线程A执行过程中，启动子线程B，那么线程A在启动子线程B之前对共享变量的修改结果对线程B可见。
 - 5、线程终止： 在主线程A执行过程中，子线程B终止，那么线程B在终止之前对共享变量的修改结果在线程A中可见。
 - 6、线程中断： 对线程interrupt()方法的调用先行发生于被中断线程代码检测到中断事件的发生，可以通过Thread.interrupted()检测到是否发生中断。
 - 7、传递性： happens-before原则具有传递性，即A happens-before B ， B happens-before C，那么A happens-before C。
 - 8、对象终结：一个对象的初始化的完成，也就是构造函数执行的结束一定 happens-before它的finalize()方法。

&emsp;&emsp;编译器和CPU对指令进行优化以保障程序执行更为高效、优化，所以在保障Happens-Before的基本原则的情况下，会对指令重新重新排序优化，我们这里还是主要就volatile进行讨论。
给出一个例子，在没有任何保障的情况下执行:

```java
public class VolatileTest1 {
    int i = 0;
    boolean flag = false;

    private void write() {
        flag = true;
        i = 1;
    }

    private void read() {
        if (flag) {
            System.out.println(i + 1);
        } else {
            System.out.println(i * 10);
        }
    }

    public static void main(String[] args) {
        VolatileTest1 test = new VolatileTest1();
        Thread t1 = new Thread(() -> test.write());
        Thread t2 = new Thread(() -> test.read());
        t1.start();
        t2.start();
    }
}

// 输出结果:
可能值： 0、2、10
``` 

&emsp;&emsp;来分析一波，不考虑编译器会进行指令重排序的情况下：    

- 线程t1修改 flag=true 前，输出 0 无疑；
- 线程t1修改 flag=true 同时，输出 0 无疑；
- 线程t1修改 flag=true 后，可能输出0 或者 2， 因为普通变量flag的变化后的值t2还未得到。

&emsp;&emsp;在考虑指令重排序的情况下： write方法可能变成这种：    
```java
i = 1;
flag = true;      
```

&emsp;&emsp;此时：

- 线程t1修改 flag=true 前，输出 10 无疑；
- 线程t1修改 flag=true 同时，输出 10 无疑；
- 线程t1修改 flag=true 后，可能输出10 或者 2， 因为普通变量flag的变化后的值t2还未得到。

&emsp;&emsp;所以在多线程的情况下，指令的重排序可能其它不可预期的结果，那volatile修饰的变量的就通过禁止重排序来保证程序执行有序性。具体是什么呢？那就是通过在变量读/写区域插入内存屏障来隔离重排序指令。

##### 重排序有哪些
&emsp;&emsp;指令的重排序可以分这几个类型：

![指令重排序](/images/191115_java_threads_volatile_2.png)

- 1. 编译器优化重排序：不改变单线程语言的情况下进行重排序优化，满足as-if-serial语义。存在数据依赖的不会被重排序。
- 2. 指令级并行的重排序。现代处理器采用了指令级并行技术（Instruction-Level 
Parallelism，ILP）来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。要求编译器在生成指令的时候加入相应的内存屏障。
- 3. 内存系统的重排序。由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。

#### 内存屏障

#### 原子性问题
&emsp;&emsp;volatile不保证原子性是说类似于a++操作，再举一例：

```java
public class VolatileTest2 {
    int a = 0;
    private void increase() {
        a++;
        System.out.println(a);
    }

    public static void main(String[] args) throws Exception{
        VolatileTest2 test = new VolatileTest2();
        for (int i = 0; i < 1000; i++) {
            new Thread(() -> {
                test.increase();
            }).start();
        }
    }
}

// 观察输出结果：
结果小于等于1000
```
&emsp;&emsp;想必其中缘由了解多线程的同学都是知道的，多个线程同时读取a值，然后同时做++操作，那么后面线程的++操作结果就有可能覆盖前面的++操作的结果，主要是该类操作非原子性操作，要分为多个操作步骤，由下图可见，线程2可以在对同一个无volatile修饰的变量有线程1的从主存read前至write到主存后等多个时间点，很容易理解，只有最后一个时间点读才能保证独到最新的更新数据：

  ![](/images/191115_java_threads_volatile_3.png)

&emsp;&emsp;上例中，将变量a用volatile进行修饰后，你觉得结果能是1000吗？虽然缓存一致性协议和内存屏障保证了变量a的可见性，但线程2仍然可以在线程1的多个时间点进行读取主存中的a到自己的工作空间，所以变量a仍然无法得到原子性保证:
  ![](/images/191115_java_threads_volatile_4.png)



