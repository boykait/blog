---
title: java多线程之AQS
date: 2019-11-22
categories:
  - java多线程
tags:
  - java
  - 多线程
---

1. 什么是AQS
  前面Lock锁的常规使用部分介绍到了两种比较常用的锁ReentrantLock和ReentrantReadWriteLock，这两种锁实现尤其是后者给多线程的并发访问场景的带来了不小的性能提升，我们都知道，针对同一块临界资源，在使用锁的lock()方法锁住的时候，同一时刻只能有一个线程访问，知道锁的释放unlock()，还有两种锁实现都具备公平和非公平的能力，够强吧，这一切底层都归功于AQS(AbstractQueuedSynchronizer，抽象队列同步器)，除了Lock锁，还有其它的如:Semaphore都是依赖于AQS, 此外，我们还可以借助AQS模板功能实现自己的同步器。

### AQS结构概览
&emsp;&emsp;AQS主要是对队列进行入队和出队的过程，当然考虑到多线程环境下，必须对队列操作进行同步操作，这个就需要一个状态state来进行控制，简单来说，就是谁获取了对这个状态字段的读写控制权，谁就可以对队列进行相关操作。AQS内部就是通过一个int类型的state字段来进行控制，为了实现其本身的通用性，state被一分为二，高16位表示shared，看其内部简要结构：

```java
public abstract class AbstractQueuedSynchronizer
	static final class Node {
      volatile Node prev;
	  volatile Node next;
	  volatile Thread thread;
	  Node nextWaiter; // 用于Condition条件判断
	}
	private transient volatile Node head;  // 链表头
	private transient volatile Node tail;   // 链表尾
    private volatile int state;    // 同步状态，该字段的更新是CAS操作过程完成的
}

```
### 1. 从ReentrantLock入手
&emsp;&emsp;锁是面向开发者，内部为我们屏蔽了太多细节，可重入锁ReentrantLock内部实现了内置同步器Sync，并基于该同步器继续实现了NonfairSync和FairSync两个同步器，这俩的区别主要在于新加入的线程会以抢占的方式去尝试获取锁，失败则进入同步队列，而后者是直接进入同步队列，前者性能要好一些，但是可能存在请求饿死现象（人家等得花儿都谢啦，还没轮到我，饿饿饿啊）。 以默认的实现非公平同步器为例，当我们在尝试调用lock.lock()方法时，执行如下：
```java
    // NonfairSync
	final void lock() {
		// state更新成功，表明可以独占，可加锁
		if (compareAndSetState(0, 1))
			// 设置独占线程
			setExclusiveOwnerThread(Thread.currentThread());
		else
			acquire(1);
	}

	// AQS
    public final void acquire(int arg) {
        // NonfairSync实现的tryAcquire处理失败，进同步队列，自己阻塞起来
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt(); // 核心是调用线程自身的interrupt方法将自己阻塞起来
    }

```

&emsp;&emsp;锁的另外一个核心操作-释放锁lock.unlock()，源码如下：
```java
    // NonfairSync
	public void unlock() {
		sync.release(1);
	}

	// AQS
	public final boolean release(int arg) {
	    // 将现在自身对应的state量去掉，成功了就从队列里面唤醒一个新的Node（线程）来执行
		if (tryRelease(arg)) {
			Node h = head;
			if (h != null && h.waitStatus != 0)
				unparkSuccessor(h);
			return true;
		}
		return false;
	}
```

&emsp;&emsp; Condition在Lock常规使用中完成生产者和消费者demo时有用到，Condition的好处能够更加精细地在并发场景中进行条件控制，可以通过await()线程自身进入等待、 signal/signalAll唤醒其它等待线程。接下来继续分析一波，看看调用过程：
```java
 // applicaiton
 lock.newCondition();
 
 // ReentrantLock 
 public Condition newCondition() {
        return sync.newCondition();
 }
 
 final ConditionObject newCondition() {
	return new ConditionObject();
}
```

&emsp;&emsp;在实例化Condition时的乾坤就是实例化了一个ConditionObject，该类是在AQS内部定义的，其实也是实现了一个FIFO队列，该队列中依次存放的就是下一个需要被唤醒执行的Node(线程), 那么既然我们可以调用多次newCondition(),那么就意味着可以产生多个ConditionObject实例，对应着可就产生多少个等待队列，瞅一眼ConditionObject的定义：
```java
public class ConditionObject implements Condition {
	private transient Node firstWaiter;
	private transient Node lastWaiter;
	// 条件等待入队
	public final void await()
	// 唤醒
	public final void signal()
	public final void signalAll()
	// 其它await方法
	...
}
```
&emsp;&emsp;那就其实不用写太多了，await和sigal系列无非就是入队和出队的过程啦，感兴趣的同学可以自己去研究一波。

#### 共享和独有模式是如何实现的？
&emsp;&emsp;在可重入读写锁中实现了多线程实现共享读，互斥写，这里面又有啥乾坤呢？首先来说，可重入读写锁内部存在两把锁：读锁(ReadLock)和写锁(WriteLock)，这两者实现内部都实现了独特的同步器，两者友好协商处理AQS的state状态。
&emsp;&emsp;看看读锁，读的特点就是共享，所以调用了AQS另一系列操作接口，主要是带shared，一看就明白：
```java
  lock.lock() -> sync.acquireShared(1);
  lock.lockInterruptibly() -> sync.acquireSharedInterruptibly(1);
  lock.tryLock() -> sync.tryReadLock();
  lock.tryLock(timeout, unit) -> sync.releaseShared(1);
  lock.unlock() -> sync.releaseShared(1);
  lock.newCondition() -> 不支持
```

&emsp;&emsp; 以读锁的lock.lock()方法切入进行分析：

```java
public final void acquireShared(int arg) {
	if (tryAcquireShared(arg) < 0)
		doAcquireShared(arg);
}
```

&emsp;&emsp;进入acquireShared方法,翻译一下主要干的事情：
```java
1. 写锁被其它线程持有，直接失败；
2. 否则根据队列策略判断是否应该阻塞，如果不是，则可以通过CAS操作更新state值；
3. 如果2步骤失败则自旋重试。
```

```java
protected final int tryAcquireShared(int unused) {       
	Thread current = Thread.currentThread();
	int c = getState();
	if (exclusiveCount(c) != 0 &&
		getExclusiveOwnerThread() != current)
		return -1;
	int r = sharedCount(c);
	if (!readerShouldBlock() &&
		r < MAX_COUNT &&
		compareAndSetState(c, c + SHARED_UNIT)) {
		if (r == 0) {
			firstReader = current;
			firstReaderHoldCount = 1;
		} else if (firstReader == current) {
			firstReaderHoldCount++;
		} else {
			HoldCounter rh = cachedHoldCounter;
			if (rh == null || rh.tid != getThreadId(current))
				cachedHoldCounter = rh = readHolds.get();
			else if (rh.count == 0)
				readHolds.set(rh);
			rh.count++;
		}
		return 1;
	}
	return fullTryAcquireShared(current);
        }
```

&emsp;&emsp;在do中主要是完成：

```java
private void doAcquireShared(int arg) {
	final Node node = addWaiter(Node.SHARED);
	boolean failed = true;
	try {
		boolean interrupted = false;
		for (;;) {
			final Node p = node.predecessor();
			if (p == head) {
				int r = tryAcquireShared(arg);
				if (r >= 0) {
					setHeadAndPropagate(node, r);
					p.next = null; // help GC
					if (interrupted)
						selfInterrupt();
					failed = false;
					return;
				}
			}
			if (shouldParkAfterFailedAcquire(p, node) &&
				parkAndCheckInterrupt())
				interrupted = true;
		}
	} finally {
		if (failed)
			cancelAcquire(node);
	}
}
```


