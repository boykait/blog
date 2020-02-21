---
title: Java线程池（一）：运行阶段可以修改参数吗
date: 2020-02-19
categories:
  - Java
tags:
  - Java
  - 多线程
---

> 知道这个问题的朋友可以跳过这篇水文，之前同司的朋友年前做了一波冲刺，挑战了蚂蚁大厂，前些天有时间我俩聊起这事，感觉大厂确实还是有点东西，问的东西很细，又比较偏，我不禁唏嘘，啊啊，其实更多的是映射出自己低水平，那么其中就有一个问题问到线程池运行阶段如果修改核心线程数比如将5个改成3个？其实没怎么研究过线程池的朋友可能有点蒙，线程池使用得好好的，没考虑过更改问题，莫不然是高大上的问题？看了这篇文章，你会知道，额，原来就只这么会事嘛。

#### 示例
我默认大家基本知道线程池的使用了，至少知道创建线程池的几个参数是干嘛的，不拐弯子，其实线程池本身就支持在运行过程中动态修改参数的，纳尼？我这直接奉上测试Demo:

```java
/**
 * @author boykait
 * @version 1.0
 * @date 2020/2/20 19:51
 */
public class ExecutorTest {
    public static void main(String[] args) throws Exception{
        int cpuSize = Runtime.getRuntime().availableProcessors();
        // 创建线程池
        ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(
                1,
                cpuSize * 4,
                30,
                /*可以指定外部ThreadFactory*/
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(200),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );
        // 打印修改之前参数
        System.out.println("修改前：");
        System.out.println("核心线程数：" + poolExecutor.getCorePoolSize());
        System.out.println("最大线程数：" +poolExecutor.getMaximumPoolSize());
        System.out.println("线程池大小：" +poolExecutor.getPoolSize());
        System.out.println("达到过的最大值：" +poolExecutor.getLargestPoolSize());
        System.out.println("正在使用的线程数：" +poolExecutor.getActiveCount());
        System.out.println("任务数：" +poolExecutor.getTaskCount());
        System.out.println("完成的任务数：" + poolExecutor.getCompletedTaskCount());
        System.out.println("阻塞队列：" +poolExecutor.getQueue());
        System.out.println("拒绝策略：" +poolExecutor.getRejectedExecutionHandler());
        System.out.println("线程工厂：" +poolExecutor.getThreadFactory());
        poolExecutor.execute(() -> {
            // 执行修改
            poolExecutor.setCorePoolSize(10);
            poolExecutor.setMaximumPoolSize(cpuSize * 5);
            poolExecutor.setKeepAliveTime(60, TimeUnit.SECONDS);
            poolExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());
        });

        // 等一会不然打印看不出效果
        Thread.sleep(10000);
        // 打印修改结果
        System.out.println("修改后：");
        System.out.println("核心线程数：" + poolExecutor.getCorePoolSize());
        System.out.println("达到过的最大值：" +poolExecutor.getMaximumPoolSize());
        System.out.println("线程池大小：" +poolExecutor.getPoolSize());
        System.out.println("最大线程数：" +poolExecutor.getLargestPoolSize());
        System.out.println("正在使用的线程数：" +poolExecutor.getActiveCount());
        System.out.println("任务数：" +poolExecutor.getTaskCount());
        System.out.println("完成的任务数：" + poolExecutor.getCompletedTaskCount());
        System.out.println("阻塞队列：" +poolExecutor.getQueue());
        System.out.println("拒绝策略：" +poolExecutor.getRejectedExecutionHandler());
        System.out.println("线程工厂：" +poolExecutor.getThreadFactory());
        // 关闭线程池
        poolExecutor.shutdown();
    }
}
```    
执行结果：

```    
修改前：
核心线程数：5
最大线程数：32
达到过的最大值：0
任务数：0
完成的任务数：0
阻塞队列：[]
最大线程数：0
正在使用的线程数：0
拒绝策略：java.util.concurrent.ThreadPoolExecutor$CallerRunsPolicy@1540e19d
线程工厂：java.util.concurrent.Executors$DefaultThreadFactory@677327b6
修改后：
核心线程数：3
最大线程数：40
达到过的最大值：1
最大线程数：1
正在使用的线程数：0
任务数：1
完成的任务数：1
阻塞队列：[]
拒绝策略：java.util.concurrent.ThreadPoolExecutor$DiscardPolicy@7ba4f24f
线程工厂：java.util.concurrent.Executors$DefaultThreadFactory@677327b6    
```

ok，线程池运行中，初始化所设置的几个参数均是可以更改的，先记住这点。那么，这个是直接更改就可以了吗？线程池框架在实现时考虑到很多的场景，不然我5个核心线程都在用，你给我cut掉2个至少要给个交代啊？接下来我们分析一下这几个参数的更改必要操作。

#### 调整核心线程数
上硬核代码：
``` java
public void setCorePoolSize(int corePoolSize) {
    if (corePoolSize < 0)
        throw new IllegalArgumentException();
    int delta = corePoolSize - this.corePoolSize;
    this.corePoolSize = corePoolSize;
    // case 1. 如果当前正在使用的核心线程数多余修改后的核心线程数，中断一部分线程
    if (workerCountOf(ctl.get()) > corePoolSize)
        interruptIdleWorkers();
    else if (delta > 0) {
    // case 2. 如果是增大核心线程的数量，视情况则增加工作线程
        int k = Math.min(delta, workQueue.size());
        while (k-- > 0 && addWorker(null, true)) {
            if (workQueue.isEmpty())
                break;
        }
    }
}
```
细说一下，首先，核心线程数量corePoolSize的数量肯定会设置成为我们想要的数值，就case 1来说，调用interruptIdleWorkers来中断线程，但是中断是有条件的，如果当前线程在执行任务，此时是不可中断的，我理解这样的线程最终会在keepAliveTime时间内结束处理的任务，不管有没有正确完成，在后面的某一个时间点内核心线程会调整到我们具体设置的值上:

```java
private void interruptIdleWorkers() {
    interruptIdleWorkers(false);
}

private void interruptIdleWorkers(boolean onlyOne) {
    // 只可有一个线程操作
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 循环线程集合，将处于空闲状态的线程中断掉
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}

// 其中workers定义：
private final HashSet<Worker> workers = new HashSet<Worker>();
```
如果是调大核心线程数，比如由5个调整到10个，框架并不是立马又启动5个线程，而是结合观察阻塞队列里面的任务数，根据代处理的任务数来创建新的worker线程：如果阻塞队列里面有两个任务代处理，那么会新增两个核心线程，如果为0个，那一个都不创建。，

#### 设置其它参数
其实，通过分析设置设置corePoolSize，其它的就非常之简单了，看看一下就明白了：   
 
```java
// 设置 maximumPoolSize
public void setMaximumPoolSize(int maximumPoolSize) {
    ...
    this.maximumPoolSize = maximumPoolSize;
    if (workerCountOf(ctl.get()) > maximumPoolSize)
        interruptIdleWorkers();
}

// 设置 keepAliveTime
public void setKeepAliveTime(long time, TimeUnit unit) {
    long keepAliveTime = unit.toNanos(time);
    long delta = keepAliveTime - this.keepAliveTime;
    this.keepAliveTime = keepAliveTime;
    if (delta < 0)
        interruptIdleWorkers();
}

// 设置拒绝策略 rejectedExecutionHandler
public void setRejectedExecutionHandler(RejectedExecutionHandler handler) {
    this.handler = handler;
}

// 指定线程工厂
public void setThreadFactory(ThreadFactory threadFactory) {
    this.threadFactory = threadFactory;
}
```

#### 总结
到此，所以当问到线程池运行过程中可以修改参数吗？可以！！！ 1，2，3...
需要需要知道：
- 那些参数可以修改？
- 修改核心线程数(其它相对简单)？