layout: GC算法
title: GC算法
date: 2017-02-19 17:57:45
tags: 技术
cover: http://a0.att.hudong.com/07/01/31300543122909143996016035883.jpg
---
Java相比较与其它语言最大的特性之一，就在于它提供了GC。开发者不必为了一些内存空间纠结，只需要不停的敲的业务代码就行了（虽然有点小悲哀）。但是问题来了，对于一些初级程序员来说，GC仅仅发生在黑盒中，根本无法感知一些正在发生的问题。但是作为一个老司机，必须对这一个问题有着深刻的理解。包括GC中STOP THE WORLD的危害，以及如何通过GC日志来排查问题。

来源：https://www.cnblogs.com/ityouknow/p/5614961.html

### 垃圾收集算法：
1. 标记清除算法
2. 复制算法
3. 标记压缩算法
4. 分带收集算法

### 收集器（具体实现）
* serial收集器 (串行收集器：新生代复制，

老年代复制压缩) -XX:+ UseSerialGC 
* parNew收集器（多线程版本serial收集器，老年代串行，新生代并行）  -XX:+UseParNewGC
* parallel收集器
* cms(Concurrent Mark Sweep) 
    * initial mark -> stop the world
    * concurrent mark -> GC Root Tracing
    * remark -> stop world
    * concurrent sweep
* G1垃圾收集器
    * 堆不分区，有利于大对象
    * 预测时间

### 流行的几种组合：
* serial
* ParNew + CMS
* ParallelYoung + ParallelOld
* G1GC
