---
title: Java线程池（三）：线程池执行的原理
date: 2020-02-22
categories:
  - Java
tags:
  - Java
  - 多线程
---
> 无论是为了面试准备或是实际项目的运用，理解Java线程池的原理都是都是很有帮助的，因为项目中发现一些参数为什么这么设置或者不合理的设置，由于我们的堆线程池的不熟悉，可能造成项目整体的质量下滑，比如CPU使用率飙升，内存OOM什么的，好的，不装bi，虽然是面向搜索引擎写作，但是也算是一个总结，所以，在这整理一篇多线程的执行原理，以对这个知识点的一个学习小结。

#### execute提交任务做了什么事
这个问题，有一张很经典的图可以说明execute的执行流程：    
![线程池执行流程图](/images/200223_java_thread_executors_1.png)
看一下execute方法执行逻辑，比较清晰，一目了然:
```java
public void execute(Runnable command) {
    int c = ctl.get();
    // case1. 创建核心线程
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // case2. 
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 入队过程中发现线程池不在运行状态，移除任务并让饱和策略进行处理
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 如果没有运行中的线程，则新创建一个来处理
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // case3. 创建非核心线程，超标了则执行饱和处理逻辑
    else if (!addWorker(command, false))
        reject(command);
}
```

reject没什么好说的，就是调用具体的拒绝处理策略就完事了，主要就是创建工作线程addWorker了。

```java
 private boolean addWorker(Runnable firstTask, boolean core) {
        // for循环的主要目的就是判断是否线程数的限制，如果满足则修改线程池中工作线程数量，否则返回
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 实例化一个工作线程
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // 加入工作队列
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                // 添加新工作线程并启动内部的线程
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                // 如果处理失败了，修改workcount数量并将工作线程从队列中移除(如果已经创建了工作线程，但没有正确启动)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

其实逻辑同样比较不难:    
![线程池执行流程图](/images/200223_java_thread_executors_2.png)

我们需要知道Worker到时是什么东西，为什么调用它内部线程的start方法就能执行我们的任务了呢？先看一下Worker类的定义，实现Runnale了说明线程启动后就会调用Worker的run方法，在newThread指定Runnable为当前的worker对象，就能够调用该worker实例对象的run方法了：
```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
 {
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        // 持有任务
        this.firstTask = firstTask;
        // 从线程池中实例化内部具体操作线程，
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
  }
```

剩下的事情就交给runWorker方法了，其实整个worker的生命周期在runWorker方法结束的时候就结束它的功用了，worker主要的功能就是在runWorker中以阻塞的方式读取任务，执行任务，如果阻塞队列中没有活儿了，那么自身就会在配置的keepAliveTime到达后结束：

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // 循环获取执行任务
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                // 执行前钩子函数
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 调用执行我们的业务逻辑
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // 执行后钩子函数
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // 善后处理，比如从工作线程修改workCount数、workers中移除当前的worker对象以及保证最少可用线程为1等
        processWorkerExit(w, completedAbruptly);
    }
}
```

ok，submit底层是调用的execute，看了这个你或许就会知道，哦，原来execute方式是可以获取到执行结果的，因为说白了submit方式就是对execute一个简单的封装。
```
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}

public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}

public <T> Future<T> submit(Callable<T> task) {
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

#### 总结
又到总结时刻，我们应该知道：

- 线程池的执行流程；
- 线程的核心方法execute的执行过程；
- 知道execute和submit两者的关系
当然线程池还有很多需要继续研究的比如参数设置成什么样子比较合理，如何优雅关闭线程池等，下波见。