---
title: AQS源码理解
date: 2018-01-28 00:00:01
tags: 并发
toc: true
cover: http://a0.att.hudong.com/07/01/31300543122909143996016035883.jpg
---
很多人对于Java线程的并发都是停留在Lock层面，通过Lock能够将synchronized粗大的颗粒划分为很小的颗粒度。然而很多人可能并没有真正去了解Lock的实现原理。
而不妨一说，AQS就是依靠数据结构的FIFO queue和compareAndSet来现实了强大的并发控制。那么今天就让我们一起来“解剖”AQS，从数据结构角度入手，再到具体实现，一览AQS的全貌。

<!--more-->

## AQS功能介绍
有些同学可能对AQS不太熟悉，那么先介绍一下。AQS(AbstractQueuedSynchronizer.class)是ReentrantLock、CountDownLatch等并发工具实现的父类，由AQS来定义了谁能拿到资源、谁需要等待，子类负责抢夺顺序的实现。

AQS提供了两种锁：独占锁和共享锁

## Node
我们先来看第一个数据结构Node。Node将Thread抽象成Node，同时赋予Node状态，用不同的状态来控制Thread的park和unPark。
所以在这一节，你需要了解Node的不同状态，请看下面摘录的源码。

```java
static final class Node {

    /** 
     * 下面四种都为Node的状态
     * CANCELLED 表明线程取消
     * SIGNAL 表明成功拿到锁的线程需要唤醒
     * CONDITION 表明线程按照定义的条件等待
     * PROPAGATE 表明下一个acquireShared无条件propagate（在共享锁中使用）
     */
    static final int CANCELLED =  1;

    static final int SIGNAL    = -1;
    
    static final int CONDITION = -2;

    static final int PROPAGATE = -3;

    // ------------------------------------------------ //

    volatile int waitStatus;

    volatile Node prev;

    volatile Node next;

    // 关联线程
    volatile Thread thread;

    Node nextWaiter;
}
```

## AbstractQueuedSynchronizer
再来看AbstractQueuedSynchronizer的数据结构，很明显是一个典型的双向链表，同时使用了state来控制并发。

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    /**
     *      +------+  prev +-----+       +-----+
     * head |      | <---- |     | <---- |     |  tail
     *      +------+       +-----+       +-----+	
 	 */

    private transient volatile Node head;

    private transient volatile Node tail;

    /**
     * 抽象出来的资源
     */
    private volatile int state;
}
```

## ReentrantLock实现 EXCLUSIVE（独占锁）
以ReentrantLock为例，我们来看下如何使用AbstractQueuedSynchronizer。

1. Sync实现AbstractQueuedSynchronizer，定义lock为抽象方法，派生出FairSync和NonfairSync。
FairSync.lock加入争夺队列，NonfairSync直接去拿资源，如果有线程正在使用，才加入队列
2. 调用ReentrantLock.lock的时候去调用Sync.lock

### lock过程
以FairSync为例，lock 等价于 acquire(1);

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
看上面的代码，其实分为了两步：
1. tryAcquire(arg) --> FairSync自己实现
    // 查看前面是否有竞争队列
2. acquireQueued(addWaiter(Node.EXCLUSIVE), arg)
    // 加入队列，模式是独占锁

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        /**
         * 注意这里自旋去获取锁
         */
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
          	/**
	         * 会去检查前面的Node的状态，当满足一定条件后才会将线程Park住，注意这时没有跳出循环
	         */
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
    	** //Tips: 注意这里没有Catch，在并发包很多类中都有这样的用法，可以google看看**
        if (failed)
            cancelAcquire(node);
    }
}
```

### unlock 过程

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /**
     * 成功后将后面的node转为unpark
     */    
    Node s = node.next;
    // 存在后面的node有别的状态的情况，不符合要求则继续往后找
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
**Tips：建议大家看看LockSupport，注意区分LockSuppork.park和Thread.interrupted概念上的不同。**
java线程阻塞中断和LockSupport的常见问题: http://agapple.iteye.com/blog/970055

## CountDownLatch实现 SHARD（共享锁）
说完了独占锁，我们来看看共享锁，以CountDownLatch为例（CountDownLatch的实现比较简单便于理解，如果想更好的使用AQS可以看看ReentrantReadWriteLock的实现）。

先说下CountDownLatch使用方式：

1. CountDownLatch(int n);
2. 工作线程调用CountDownLatch.countDown()，减少步骤1中的n
3. 需要共享的线程调用CountDownLatch.await，当1中的n为0的时候，线程才执行。这里可以是多个线程同时使用await，这里就是共享锁的使用场景之一。

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                	/**
                	 * 这里和独占锁不同
                	 */
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
看完代码是不是觉得和独占锁基本一毛一样。对，除了setHeadAndPropagate(node, r)这个。

setHeadAndPropagate的作用：
当一个shard的Node获得了资源，那么就会唤醒队列中他之后的连续的shard节点，使其同时运行。
这就是共享锁和独占锁不同之处。

## 总结一下
AQS提供了子类接触资源的方式，同时不同的唤醒方式，提供给了使用者独占或者共享等不同的锁的使用方式。
这里不得不再次感叹数据结构之美妙，一个简简单单的Queue就玩出了独占、共享锁等等花样。所以，理解源码一个很好的方式就是读懂它内在的数据结构。
折腾不止，学习不止。
望与君共勉~

## 资料来源
 AQS源码分析之独占锁和共享锁： http://blog.csdn.net/luofenghan/article/details/75065001
 JDK源码AQS: http://childe.net.cn/2017/02/14/JDK%E6%BA%90%E7%A0%81-AQS/