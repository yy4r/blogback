---
title: zookeeper
date: 2017-02-19 17:49:32
tags: 技术
cover: http://a0.att.hudong.com/07/01/31300543122909143996016035883.jpg
---
## Watcher接口
实现Watcher的类是一个回调函数，process中实现的方法，会在znode变更的时候触发
```java
interface Watcher() {
// 不太懂为啥在Wacher中又定义了一个interface
    public interface Event(){        
        public enum EventType{}
    }
    abstract public void process(WatcherEvent event){}
}
```
<!-- more -->
调用zookeeper.create的时候要求传入权限控制参数（ACL），找了文章发现在UNIX中权限的管理还是很有意思的：
http://ifeve.com/zookeeper-access-control-using-acls/ 
有空可以继续研究一下

## 继续看zookeeper
发现在构造函数的实现中，有两个Thread。这个就有意思了，为啥在构造函数中有Thread？可以继续追踪下去 
```java
SendThread sendThread = new SendThread(clientCnxnSocket); 
ReadThread eventThread = new EventThread();
```
查阅资料可以了解到zookeeper实现了一个事件驱动模型，先贴一张图或许有助于理解：

![来源http://www.wangjingfeng.com/38.html](https://static.oschina.net/uploads/img/201702/05224610_HZKB.gif "在这里输入图片标题")

来源http://www.wangjingfeng.com/38.html

```java
class sendThread {
    void run() {
        // 实现NIO
        // pendingQueue等待响应的packet，outgoingQueue等待发送的packet
        //int waitTimeOut, List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue,ClientCnxn cnxn
        clientCnxnSocket.doTransport(waitTimeOut, pendingQueue, outgoingQueue,cnxn){};
    }   
}
```
### EventThread -> BlockedQueue -> SendThead
通过加锁固定长度的队列实现消费者模型，同时sendThread使用了java自带的NIO技术，每次轮询所有的selector，处理已经传输完成的selector。很好的实现了解耦。
## 一些其他class

```java
class ClientCnxn {
    // ClientCnxn是一个管理I/O的类， 包含了一系列的可用的servers，并且控制这些servers的开关
    
} 
```
```java
class AuthData {
    String scheme;
    byte data[];
}
```
```java
static class Packet {
    RequestHeader requestHeader;
    ReplyHeader replyHeader;
    Record request;
    Record response;
    ByteBuffer bb;
    AsyncCallBack cb;
}
```

```java
class EventThread {
    void queueEvent(WatcherEvent watcherEvent){}
    void queuePacket(Packet packet){}
    void processEvent(Object event){
        WatcherSetEventPair pair = (WatcherSetEventPair) event;
        watcher = pair.watchers;
        watcher.process(pair.event);
    }
}
```

```java
public class ClientCnxnSocketNio{
    
}
 ```