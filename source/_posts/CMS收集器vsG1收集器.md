---
title: CMS收集器 VS G1收集器
date: 2018-03-11 19:40:41
tags: JVM
toc: true
cover: http://a0.att.hudong.com/07/01/31300543122909143996016035883.jpg
---
最近公司，将线上的GC逐步配置成了G1收集器，同时Java9中G1也成为了默认的收集器。所以作为一个程序员，不得不了解下G1到底是个什么结构，凭什么他能够干掉CMS收集器。

<!-- more -->

## CMS收集器
既然是CMS收集器和G1收集器的比较，那么的先来讲清楚CMS收集器是个什么东西。CMS收集器，英文名是Concurrent Mark Sweep Garbage Collection（并发标记垃圾清扫收集器），其实你看英语名就知道CMS到底在哪里有了巨大的提升。

### CMS收集器出现前的时代
继续举那个例子，小明在卧室吃东西丢垃圾，然而小明的妈妈要定时来打扫垃圾，如果边打扫小明还继续丢垃圾，势必卧室就还是脏的，解决办法就只能是小明停止吃东西，妈妈先打扫。OK，这就是比较典型的serial收集器，parallel收集器……

这种情况下，无论是单线程还是多线程，他们都需要小明（或者说是jvm）停止生产垃圾，也就是STOP THE WORLD。

在更规范一点的讲，因为serial收集器，parallel收集器都是靠着可达性分析来判断，是否将这个对象标记为可回收的对象。看下面一张图：

<img src="https://olwr1lamu.qnssl.com/Animation_of_the_Naive_Mark_and_Sweep_Garbage_Collector_Algorithm.gif"  alt="Animation_of_the_Naive_Mark_and_Sweep_Garbage_Collector_Algorithm"/>


里面存在一个最大的问题，遍历和标记是在一起的。也就是说，它遍历所有的节点，同时边遍历边标记。那么导致jvm需要停止很长的时间。所以解决方案就是能否将标记和遍历分离，减少STW的时间。OK，这就是Concurrent Mark Sweep Garbage。 

## CMS的原理
CMS的原理就是传说中的三色标记算法。（其实很简单的，慢慢来看）

<img src="https://olwr1lamu.qnssl.com/Animation_of_tri-color_garbage_collection.gif"  alt="Animation_of_tri-color_garbage_collection"/>


> 三色标记算法是对标记阶段的改进，原理如下：

1. 首先创建三个集合：白、灰、黑。
2. 将所有对象放入白色集合中。
3. 然后从根节点开始遍历所有对象（注意这里并不递归遍历），把遍历到的对象从白色集合放入灰色集合。
4. 之后遍历灰色集合，将灰色对象引用的对象从白色集合放入灰色集合，之后将此灰色对象放入黑色集合
5. 重复 4 直到灰色中无任何对象
6. 重新标记步骤4，5期间产生的对象（go 中使用）
7. 收集所有白色对象（垃圾）

来看下CMS(Concurrent Mark Sweep) 收集器具体的实现：

1. initial mark -> stop the world
2. concurrent mark -> GC Root Tracing
3. remark -> stop world
4. concurrent sweep

**步骤1**表示遍历Root set，将对象放入灰色区域，对应三色标记的3
**步骤2**表示遍历灰色区域的对象，但这时是并发进行的，是不会造成STOP THE WORLD，对应的是三色标记4，5
**步骤3**是重新标记在步骤2中，同步进行所遗漏的对象
**步骤4**将没有标记的对象，全部清除

## G1收集器

### 介绍
先来看下G1官网对G1的介绍

> The Garbage-First (G1) collector is a server-style garbage collector, targeted for multi-processor machines with large memories. It meets garbage collection (GC) pause time goals with a high probability, while achieving high throughput. The G1 garbage collector is fully supported in Oracle JDK 7 update 4 and later releases. The G1 collector is designed for applications that:

>1. Can operate concurrently with applications threads like the CMS collector.
2. Compact free space without lengthy GC induced pause times.
3. Need more predictable GC pause durations.
4. Do not want to sacrifice a lot of throughput performance.
5. Do not require a much larger Java heap.

G1是一个为了大内存设计的多线程处理的垃圾回收器。它能满足大吞吐量的时候，GC停止时间的要求。C1完全支持JDK 7 update 4 及以后的 JDK版本。G1是为以下几种需求设计：
1. 能够和应用线程一起运行
2. 为了减少暂停时间，能够进行空间合并，而且不需要长时间的GC
3. 能够预测GC时间
4. 不影响吞吐性能
5. 不要求更大的java堆

### 与CMS收集器区别
相比于CMS是不是更符合现在这个互联网时代，动辄16G，32G内存的服务器。so，一起来看下区别：

<img src="https://olwr1lamu.qnssl.com/g1heap.png"  alt="G1heap"/>
可以看到，和CMS收集器完全不同，G1中没有固定的年轻代，老年代的区别，G1将heap直接分为了一个个小的region。这样就可以用更小的颗粒度来控制heap，同时能够使用更好的压缩使用空间。

### Young GC
<img src="https://olwr1lamu.qnssl.com/after.jpg"  alt="Young GC"/>
可以看到，相比于CMS收集器，G1收集器多了一步压缩。能够将一些没清理的对象，统一放到一个region中，这样减少了下一次GC的成本。

**需要注意的是：无论什么收集器的Young GC 都会SOTP THE WORLD，包括CMS和G1的。**

### Old Generation Collection with G1

发现网上的资料比较少，而且还缺少比较统一的说法，为了不误导大家，把官网的介绍给搬了过来。
<img src="https://olwr1lamu.qnssl.com/G1%E6%B8%85%E7%90%86%E8%BF%87%E7%A8%8B.png"  alt="G1清理过程"/>

可以看到G1也是和CMS一样使用了三色标记算法，然而不同的是G1使用了SATB，以及RSET等辅助工具来帮助进行GC。同时还增加了重置region的步骤。

## 总结
1. G1用更小的颗粒度来控制heap，同时能够通过移动region来压缩heap
2. 老年代的GC本质上都是三色标记算法，通过分离标记和遍历，来达到多线程并行的效果
3. G1还有更多的细节需要理解，包括如何控制GC时间，如何优化三色标记算法。利用空余时间继续研究jvm~


## 参考资料
1. 一张图了解三色标记法 http://idiotsky.me/2017/08/16/gc-three-color/
2. Golang 垃圾回收剖析 http://legendtkl.com/2017/04/28/golang-gc/
3. 官方介绍 http://www.oracle.com/technetwork/tutorials/tutorials-1876574.html

