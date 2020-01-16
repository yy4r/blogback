---
title: MQ的Producer理解
date: 2017-08-13 14:33:12
tags: 技术
toc: true
cover: http://a0.att.hudong.com/07/01/31300543122909143996016035883.jpg
---
好久没写文章了，最近都是一直996，难得有机会空下来写点东西。事情的缘由是在一个开发群里面有人问了个问题:
> 他是做网吧系统的，如果给每个网吧一个队列，上千个队列，RocketMQ是否能够支持？

虽然平时自诩对MQ的理解还是可以的，但是这么一个问题好像我对MQ的理解仅仅是名词上的理解，所以赶紧打开ide重新看下MQ的源码。

## 整体结构
<img src="https://olwr1lamu.qnssl.com/%E6%95%B4%E4%BD%93.png" alt="整体图"/>

先从整体上理解MQ的组成，主要有Producer、Namesrv、Broker、Consumer组成。而Producer、Broker、Consumer之间的通信主要依靠Netty实现。整体的架构看似简单，但是真的去阅读源码，才发现MQ在可靠性以及扩展性上提供了很好的支持，这才是它作为一款生产级产品最大的特色。
可靠性体现在MQ的至少一次投递，以及能够追溯消息。而这次我们可以通过Producer来粗窥MQ的扩展性。

## Procuder如何发送消息
<img src="https://olwr1lamu.qnssl.com/producer.png" alt="Procuder发送过程"/>

Procuder即消息的生产者，NameSrv中取获得当前Topic的相关信息。这里没有画出来的是，Procuder在内部缓存了一份TopicTable，只有当缓存失效时才去NameSrv获取。
获取到的TopicInfo中包含了一个List<Queue>, 这个东西很值得思考。在代码中可以看到，每一次消息投递，都对List<Queue>做了一个取余的处理。

那么就提出了一下几个问题：
***
1. 这个Queue是什么东西？
2. 这个Queue有什么用？
3. 为何要维护多个Queue呢？搞一个不就行了吗？
***

打开类一看，长这个鸟样。
```JAVA
public class MessageQueue implements Comparable<MessageQueue>, Serializable {
    private String topic;
    private String brokerName;
    private int queueId;
}
```

查阅了相关资料，来自https://jaskey.github.io/blog/2016/12/19/rocketmq-rebalance。
可以看到通过让不同的Q分布在broker上实现均衡负载，可以让某些broker来负担多一点的Q，也可以平衡的分布Q。不必让消息绑定在Broker上，而是让消息于Q相绑定，用更小的颗粒度去控制消息。同时Q不保存消息的具体内容，而是能通过Q去追溯在Broker的位置。这一招厉害呀，扩展性瞬间提高了一大截。

## 那个的问题
说回那个同学的问题，他应该是直接使用了上千的Topic，但是对于MQ来说，不同于Kafka，MQ能够支持上万的Topic(详情参见官方MQ于kafka的比较）。那么MQ是如何管理所拥有的Topic呢？下一次来分析下Broker的源码。