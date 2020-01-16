---
title: Solr AND SolrCloud 学习记录
date: 2018-05-03 00:00:01
tags: 搜索
toc: true
cover: http://a0.att.hudong.com/07/01/31300543122909143996016035883.jpg
---
最近开始负责其对接Solr的业务，发现除了Solr的语句以及增量全量的方式，还需要很多细节点需要自己去关注。因为搜索组方面除了保证提供稳定性，还需要针对不同的业务场景提供不同的解决方案，因此了解Solr能够解决些什么问题，还是非常重要的。
<!-- more --> 

## SolrCloud
除了Solr的简单查询语句，对于使用者所必须理解的就是SolrCloud的整体架构。首先需要区分两个概念：

- shard
- replica

先从名词定义上来理解，shard指的是分片，replica指的是复制品。那么比较好理解了，当索引增长到一定的量，单台机器已经不能承载那么多的索引，必须将索引拆成多分，这是你就需要好多的shard的。**shard**是将索引切分成多分索引，来实现SolrCloud的可扩展性。

而都知道分布式系统最需要的CAP理论，对于C也是必须的。所以，每一份shard都必须要有多分**replica**，这样才能够保证即使有一台服务器挂了，还是能够对外提供服务。
对于有多分replica，如果要提升处理能力，肯定就需要选举出一个**Leader**来处理逻辑。所以，在replica中就有了Leader的概念。

<img src="http://dl2.iteye.com/upload/attachment/0110/8781/f79e81a8-ae9c-381b-9d26-adb80f3d16fc.png" alt="solr shard结构"/>

### 增加索引的过程

- 用户提交Doc到任意replica
- replica会将请求转发到当前的Leader
- Leader将更新结束的Doc分发到各个replica

### 查询过程
与增加索引相对应的是查询的过程：

- 用户将查询请求发送到任意replica
- replica会将请求分解成多个子查询，分配到不同的shard的replica上(注意，这里是不需要经过Leader，每个节点都保存了所有节点信息)
- replica整合所有得到的结果，返回用户

<img src="http://dl2.iteye.com/upload/attachment/0110/8787/95641581-1018-3963-9bbf-eb85033106e7.jpg" alt="增加索引的过程"/>

### tLog
这里还需要介绍下tlog。在Lucene是没有soft Commit这个概念的，只有Hard Conmmit。而Solr Cloud为了保证，写入的索引能立即可见，提出了soft Commit。将索引先写到memory中，搜索的时候是memory 和 disk合并的结果。是不是和LSM Tree有点类似 ~
tlog就是为了软提交所存在，当索引还没内写入到硬盘中，这是断电了索引不就丢了。所以，每一次soft Commit都是先写入tlog，在写入到memory中。当然tlog还有其他大用，在我司的业务中，每天晚上的凌晨会对solr进行全量，来对索引的merge进行优化。这个时候就存在，增量丢失，而通过再次消费tlog中的日志，就能保证增量的数据不丢失。

### 故障恢复过程 AND 新replica
SolrCloud提供了两种恢复方式:
- 对等同步
- 快照复制

前者用于最近的部分丢失，而后者是由于处于脱机导致不能同步，需要将所有信息同步。而且更强大的是，无论哪种情况SolrCloud都会自动为你选择。
因此，类似的场景往往发生在需要新增一个replica来降低每台服务器的负载（使用默认的文档策略不能使不同的shard拥有不同数量的replica，但是自定义模式下可以）。这是只需要启动一个Solr实例，并将shardId赋予给它。在启动参数内添加 -Dshard=Id，那么就能够创建一个新的副本。

### 由于Solr涉及的内容较多，容笔者慢慢记录下学习的过程
内容包括：
- mmseg分词算法
- 搜索推荐、协同过滤等

<img src="https://olwr1lamu.qnssl.com/%E6%8E%A8%E8%8D%90%E7%AE%97%E6%B3%95.png" alt="推荐算法发展"/>


## 参考资料
SolrCloud之分布式索引及与Zookeeper的集成：http://josh-persistence.iteye.com/blog/2234411
Solr实战（实体书）

