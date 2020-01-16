---
title: 如何自己实现一个NIO的Server
date: 2017-04-23 16:29:58
tags: 技术
toc: true
cover: http://a0.att.hudong.com/07/01/31300543122909143996016035883.jpg
---
话说笔者最近在看一些网络通信方面的东西，恰巧看到一位大牛做了一个NIO的server的Demo。Demo虽小，但是五脏俱全，能够很好的一窥NIO服务的架构。
源码地址：https://github.com/jjenkov/java-nio-server

# 整体结构
<img src="https://olwr1lamu.qnssl.com/NIOpackge.png" width="50%" height="50%" alt="结构图"/>
整体结构如图所示，刚开始可能不知道这个Server是一个怎样的结构，但是作者很善良的给了个example，那么就先从main方法进入。

```java
Server server = new Server(9999, new HttpMessageReaderFactory(), messageProcessor);
server.start();
```
main方法中，主要初始化了项目中的Server，以及启动Server。好，那么就可以直接来看整个项目真正的启动类。

## Server
直接来看Server中的start吧，看看一个Server需要启动哪些脚手架：

```java
public void start() throws IOException {
	      // 循环创建socket放入Queue
        this.socketAccepter  = new SocketAccepter(tcpPort, socketQueue);
        
        // NIO循环遍历，查询可以进行操作的channel
        // 这里的入参都是来自Server的初始化的时候的构造函数，而且还包含一个Factory方法，这种设计类的思路可以借鉴一下
        this.socketProcessor = new SocketProcessor(socketQueue, readBuffer, writeBuffer,  this.messageReaderFactory, this.messageProcessor);
        
        // 启动线程
		    accepterThread.start();
        processorThread.start();
}
```
通过上面的类，我们可以看到，Server中主要启动了，两个线程，分别是accepterThread和processorThread。

## 架构组成
为了方便理解，我直接把整个NIO的架构给贴出来吧。
<img src="https://olwr1lamu.qnssl.com/jiagou.png" alt="NIO架构图"/>
主要包括Acceptor和SocketProcessor，两者之间通过Queue连接。

## Acceptor

```java
// 打开Socket
try{
     this.serverSocket = ServerSocketChannel.open();
     this.serverSocket.bind(new InetSocketAddress(tcpPort));
} catch(IOException e){
     return;
}
while(true){
      try{
	       // socketChannel.accept接收connection
           SocketChannel socketChannel = this.serverSocket.accept();
           // 将Socket放入队列
           this.socketQueue.add(new Socket(socketChannel));

          } catch(IOException e){
                e.printStackTrace();
          }
}
```
socketChannel可以设置为非阻塞模式，serverSocketChannel.configureBlocking(false);
这样socketChannel接收到Connoection就会立刻返回，没有则返回null。直接将socketChannel放入Channel，由SocketProcessor去负责检查Channel是否就绪。同时，需要Processor处理，返回为null的情况。

## SocketProcessor
SocketProcessor中维护了一个readSelector，其中注册了来自Queue中的Socket。
readSelector是由Java NIO包中package java.nio.channels.Selector提供。
注册分几步走：
1. 遍历Queue中的Socket
2. 初始化Socket中的IMessageReader，socketChannel中的configureBlocking设置为false（有个问题：为什么是在Procesor设置，而不是Acceptor中设置？）
3. 将Socket注册到Selector中

```java
SelectionKey key = newSocket.socketChannel.register(this.readSelector, SelectionKey.OP_READ);
key.attach(newSocket);
```
通过上面两个函数，将Socket中的socketChannel注册到了Selector中，同时返回SelectionKey。
4. 检查readSelector
5. 就绪的SocketChannel，则使用Socket初始化的messageReader，读取Message
6. 将SelectionKey从readSelector取消，并关闭SocketChannel

## Socket类
这里可以来看下作者设计的Socket这个类：
<img src="https://olwr1lamu.qnssl.com/Socket2.png" width="100%" height="75%" alt="Socket结构"/>
这里一个很好的设计是，messageRead是有Factory产生，而Factory是从messageProcessor从构造类中导入，提供了很好的扩展性。

```java
public interface IMessageReader {
    public void init(MessageBuffer readMessageBuffer);
    public void read(Socket socket, ByteBuffer byteBuffer) throws IOException;
    public List<Message> getMessages();
}
```

## 字节流
同时，在HttpUtil中还有个处理请求字节流的工具类

```java
int endIndex = HttpUtil.parseHttpRequest(this.nextMessage.sharedArray, this.nextMessage.offset, this.nextMessage.offset + this.nextMessage.length, (HttpHeaders) this.nextMessage.metaData);
if(endIndex != -1){
	Message message = this.messageBuffer.getMessage();
	message.metaData = new HttpHeaders();
	message.writePartialMessageToMessage(nextMessage, endIndex);
	completeMessages.add(nextMessage);
	nextMessage = message;
}
```
不过，知道怎么用就差不多了（其实是懒）。


# 总结
通过整个分析，可以了解SocketChannel，Selector的用法，以及对NIO Server的架构有了一个理性的认识。对于以后自己去处理NIO的一些东西，是非常有价值的，非常鼓励大家，能够去看看这个小Demo的源码。

同时，推荐两个开源项目，有利用到NIO，而且都很有意思：
Rider: https://my.oschina.net/alchemystar/blog/865237
Mycat

另外，介绍Java的一个Buffer，DirectBuffer。不同于一般的Buffer，DirectBuffer属于内存Buffer，虽然DirectBuffer的创建相较于普通Buffer耗费时间，但是对于一些大的IO，如果需要从内存 -> jvm -> 内存，DirectBuffer能够提供更高的效率。

```java
class Buffer {
	// 0 <= mark <= position <= limit <= capacity	
	int capacity,limit,position,mark;
	public Buffer flip() {
		// 将limit设置为当前position，同时position设置为0，准备开始读数据
	}
	public Buffer rewind() {
		// 将position设置为0，mark设置为-1，准备开始写数据
	}
}
```

