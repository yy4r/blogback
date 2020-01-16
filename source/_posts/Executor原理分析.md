---
title: Executor原理分析
date: 2017-02-25 00:14:51
tags: 技术
cover: http://a0.att.hudong.com/07/01/31300543122909143996016035883.jpg
---
（未写完……）
看了一篇文章，讲的是一分钟理解连接池的实现，其中讲到：连接池的关键是维护了两个数组，一个数组记录游标，另一个数组保证加锁。
链接：https://0x9.me/Zz9hp（转成了短码，有可能失效）
讲的很有意思，但是做为一个JAVAER，我对JAVA的底层如何实现“池”的概念更感兴趣，那么这一次我们就从JAVA的源码角度来看看，JAVA到底是如何实现创建线程池，拿到线程，放回线程池。
先从执行类开始说起，
```java
public interface Executor {
	//所有的Executor所继承的接口，可以发现execute是没有返回值的，所有的返回值都是保存在command（也就是Future）里面的。Executor其实就是“喂喂喂，所有的task都走这里”。
    void execute(Runnable command);
}
```
```java
public interface ExecutorService extends Executor{
	// 已经抽象出了一个Executor（以及下一步的线程池）的样子
	boolean awaitTermination(long time, TimeUnit unit);
	void shutdown();
	invokeAll(Collection<? extends Callable<T>> tasks);
}
```
在这里需要说明一下的是，在ExecutorService的实现类中，有很多有价值的CLASS：
ForkJoinPool、ScheduleThreadPoolExecutor、ThreadPoolExecutor。
还记得在看《七周七并发中》的一个很有价值的问题：
** ForkJoinPool和ThreadPoolExecutor有什么区别？**
欢迎大家自己想想。

回归正题，继续看线程池。先看下ThreadPoolExecutor的构造类。
```java
public ThreadPoolExecutor(int corePoolSize,	// 最小保持的线程数
                          int maximumPoolSize, // 最大保持的线程数
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory, // 生成Thread的工厂类
                          RejectedExecutionHandler handler // 用来处理线程池满了或者队列满了的处理类)；                     
```

```java
    static class DefaultThreadFactory implements ThreadFactory {
        private static final AtomicInteger poolNumber = new AtomicInteger(1);
        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;

        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager(); //是保证上下文切换吗？还是为了安全什么的？
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup();
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-";
        }

        public Thread newThread(Runnable r) {
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0);
            if (t.isDaemon())
                t.setDaemon(false);
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
    }
```
继续回到
```java
public class ThreadPoolExecutor {
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /**这里有三个步骤：
         * 1.如果corePool数量未满，则调用addworker原子性添加线程
         * 2.将command放入到队列，并且二次检查，检查线程的数量以及线程池是否挂掉
         * 3.如果线程数未满，则添加线程，如果线程池挂掉则退出
         */
        int c = ctl.get(); 
        /**
        * private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
        * ctl 通过位运算保存了线程数以及线程状态，有意思
        */
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
}
public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(SHUTDOWN);
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
}
```
好了，下面就是重头戏了，为了明显点特意从ThreadPoolExecutor中的execute()了出来，已经快睡着的同学可以好好看了：
```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);
            // 关闭时：检测队列是否为空
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
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```
下面是实际工作的类
```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }

        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```