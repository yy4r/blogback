---
title: ThreadLocal理解
layout: post
date: 2017-08-02 23:59
category: Java
tags: 多线程
cover: http://a0.att.hudong.com/07/01/31300543122909143996016035883.jpg
---

> 不同的线程可以通过同一个 ThreadLocal 对象获取只属于自己的数据。

## ThreadLocal.ThreadLocalMap

`ThreadLocal`的内部类。是以`ThreadLocal`的 hash 值为数组下标，`Entry`元素为值的数组。ThreadLocalMap 内部是实现了一个类似 Map 的映射关系，内部的 Entry 继承自`WeakReference<ThreadLocal<?>>`，它持有ThreadLocal的弱引用，保存`ThreadLocal.set(value)`传入的`value`。

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

## ThreadLocal

![set & get](http://image.youcute.cn/17-8-3/16513590.jpg)

### get 方法

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

### set 方法

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

## 使用地方有:

* Android的消息循环机制(Looper Handler MessageQueue)就是基于这个。
* ...

## 实例:

```java
public class Main {
    static final ThreadLocal<String> mThreadLocal = new ThreadLocal<>();
    public static void main(String[] args) {
        new Thread("thread1") {
            @Override
            public void run() {
                mThreadLocal.set("value1");
                try {
                    Thread.sleep(4000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(currentThread().getName() + " localValue:" + mThreadLocal.get());
            }
        }.start();

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
        new Thread("thread2") {
            @Override
            public void run() {
                mThreadLocal.set("value2");
                System.out.println(currentThread().getName() + " localValue:" + mThreadLocal.get());
            }
        }.start();

    }
}
```
输出:
```java
thread2 localValue:value2
thread1 localValue:value1
```

虽然是同一个`ThreadLocal对象`，而且都调用的同样的`set` `get`方法，但是`get`方法返回的值，一定是与当前线程对应的。

