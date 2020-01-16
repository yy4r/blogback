---
title: 分布式系统vs集群系统
date: 2018-12-29 17:48:21
tags: 技术
doc: true
cover: http://a0.att.hudong.com/07/01/31300543122909143996016035883.jpg
---
以前都会纠结一个问题，什么是分布式系统？什么是集群？服务器数量变多是不是就是从集群演化为分布式？

举一些例子，

1. mysql主从系统
2. mysql分库分表 + 主从系统
3. Nginx + 多台服务器构建的 Spring MVC 系统
4. dubbo的服务化系统
5. Rocketmq 双Master

在举一些被认为是分布式系统的例子，
1. Hbase
2. kafka
3. Elasticsearch
4. TIDB

<!--more-->

比较之后会发现分布式系统，有两个特点：scale（扩展） 和 failover（故障转移）。而这两点恰恰需要一个Master来做这点，而且Master也必须是可以做到failover的。

## 以kafka为例
通过对比，你可以发现分布式系统的写入和读取都有类似的地方，我们来看下kafka的读写。

### kafka写
Kafka 节点的读写流程，从 Zookeeper 找到 partition 的 Leader，同时依据配置情况，确保备份的 partition 写入完毕，才返回写入成功。

### kafka读
同时Kafka 的消费逻辑也比较搞，依据使用不同API，Offset保存在不同的位置。

配置一：创建一个Topic，保存Offset信息
配置二：保存在Zookeeper上

对比 Zookeeper 和 etcd，会发现etcd确实有很大的优势，首先是简单，Raft相比较与Paxos还是简单很多的。自己不妨看下Raft的实现方式，网上还是很有多实现手段的无论是Java还是Go。

https://gitbook.cn/books/5ae1e77197c22f130e67ec4e/index.html


实现Raft的关键：
1、保存raft节点核心数据（节点状态信息、日志信息、snapshot等），
2、raft节点向别的raft发起rpc请求相关函数
3、raft节点定时器：主节点心跳定时器、发起选举定时器。


无论是自己实现Raft还是Paxos，都需要考虑的是Master Crash的情况，这才是分布式系统最最核心的东西。