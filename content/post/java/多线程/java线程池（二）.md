---
title: Java线程池（二）：中断线程
date: 2020-02-22
categories:
  - Java
tags:
  - Java
  - 多线程
---

> 同样水文一篇，就当学习笔记，大佬自动忽略，本篇来学习一下线程中断相关知识点，alf go！

#### 中断普通线程
有过线程学习经验的小伙伴应该都知道，中断线程的几种方式：    

- 退出标志；
- stop强行终止（弃用）；
- interrupt

对应Demo:

```java
// 退出标志式
public class StopThreadTest extends Thread{
    private boolean exit = false;

    @Override
    public void run() {
        while (!exit) {
            try {
                System.out.println("别搞我，我在摸鱼");
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    public static void main(String[] args) throws Exception{
        StopThreadTest thread = new StopThreadTest();
        thread.start();
        Thread.sleep(4000);
        thread.exit = true;
    }
}

// interrupt方式    
public class InterruptThreadTest extends Thread {
    @Override
    public void run() {
        while (!isInterrupted()) {
            try {
                System.out.println("别搞我，我在摸鱼");
                Thread.sleep(5000);
            } catch (InterruptedException e) {
               break;
            }
        }
    }

    public static void main(String[] args) throws Exception {
        InterruptThreadTest thread = new InterruptThreadTest();
        thread.start();
        Thread.sleep(3000);
        thread.interrupt();
    }
}
```

#### 中断线程池中线程
使用线程池提交任务有两种：execute和submit，这两者最大的区别就是后者带有返回值的Future句柄，在[Java线程池（一）：运行阶段可以修改参数吗](./Java线程池（一）.md)中使用过execute方式提交任务，看一下submit方式提交任务：

```java
public class SubmitStopThreadTest {
    public static void main(String[] args) throws Exception {
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

        // 执行Runnable，无返回值
        poolExecutor.submit(new RunnableTask());

        // 带返回值的Runnable
        Result result = new Result("我是线程池中带返回值的Runnable骚操作");
        Future<Result> runnableFuture = poolExecutor.submit(new RunnableTaskWithResult(result), result);
        System.out.println(runnableFuture.get().getData());

        // 执行callable类型
        Future<String> callableFuture = poolExecutor.submit(new CallableTask());
        System.out.println(callableFuture.get());
        // 关闭线程池
        poolExecutor.shutdown();
    }
}

class CallableTask implements Callable<String> {

    @Override
    public String call() throws Exception {
        return "执行Callable任务";
    }
}

class RunnableTask implements Runnable {

    @Override
    public void run() {
        System.out.println("执行Runnable任务");
    }
}

class RunnableTaskWithResult implements Runnable {
    private Result result;

    public RunnableTaskWithResult(Result result) {
        this.result = result;
    }

    @Override
    public void run() {
        this.result.setData("hello 多线程： " + result.getData());
    }
}

class Result {
    private String data;

    public Result(String data) {
        this.data = data;
    }

    public String getData() {
        return data;
    }

    public void setData(String data) {
        this.data = data;
    }
}
 
//  执行结果：     
执行Runnable任务        
hello 多线程： 我是线程池中带返回值的Runnable骚操作    
执行Callable任务    
```

submit其实内部也是封装的execute方法：
```
 public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

所以execute想要得到返回值，可以通过这个方式实现，然后可以拿去futureTask中的执行结果:
```java
 FutureTask<String> futureTask = new FutureTask<String>(
    new Callable<String>() {
        public String call() {
            Thread.currentThread().stop();
            return "test";
        }
    });
poolExecutor.execute(futureTask);
```

这样停止线程可以用调用FutureTask的cancel方法：

```java
futureTask.cancel(true);

public boolean cancel(boolean mayInterruptIfRunning) {
        // 设置任务状态
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    // 调用线程的interrupt中断线程
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            // 调用finishCompletion执行任务完成，唤醒并删除所有在waiters中等待的线程
            finishCompletion();
        }
        return true;
    }
```
#### 总结

当然，我觉得线程交由线程池管理，一般不需要我们再去控制其生命周期，所以也只是探讨一下，在线程或线程池里，安全关闭一个线程的方法还是调用interrupt方法啦。
