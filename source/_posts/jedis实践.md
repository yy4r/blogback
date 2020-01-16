---
title: Jedis实践
date: 2017-02-22 18:36:55
tags: 技术
cover: http://a0.att.hudong.com/07/01/31300543122909143996016035883.jpg
toc: true
---

最近在写业务代码的时候用到的SETNX，但是在牛肉的提醒下才发现我对SETNX的理解有很多的问题，果然还是NO BI BI, SHOW ME THE CODE比较实际。

### SETNX
先讲下原理：
SETNX提供了一个锁的概念，通过判断返回值（0或者1）来执行不同的逻辑，包含了GET和SET的逻辑，在原来GET或者SET的场景直接使用SETNX就好。通过使用SETNX能够应用于业务去重，以及防止缓存穿透等场景。
一般做法：在更新缓存前，先判断SETNX是否存在，保证只有一个线程进入update（获得资源），其余的线程则是等待或者读取以前缓存的数据。
```java
// redis提供的setnx没有过期时间，从2.6.12之后set涵盖setnx功能，提供过期
// String statusCode = redis.set("lock", value, "NX", "EX", expire); 和下面使用setnx相同
String statusCode = redis.setnx("lock", value);
if( "OK" == statusCode) {
  // 成功获得锁，更新数据场景
  db.select(....);
  // 这里可能比较耗时，例如200ms
  redis.update();
  redis.del(key);
} else {
  // 没有获得锁，使用过期数据场景
  redis.get(); // or 或者一直等待
}
```
这里需要像类似我这种小白的人，说明的是Redis是单线程的，提供一个队列模型，能够保证有序的执行每个command。同时，通过各种资料查的Redis的SET和GET都是原子性的（实验方法：SET1到1000，同时GET，发现数据并没有影响）。
但是在SETNX和Del之间，会执行业务逻辑，业务逻辑并不是原子性的，在这段时间内Redis还会有多个GET。这部分数据会在大并发的时候，造成缓存穿透，有可能导致DB挂掉。
start-> SETNX -> GET -> GET -> .... ->Get->Del -> end
Get 的长度取决于业务逻辑执行时间。

SETNX只是提供了一个锁的概念，并不是存储数据（我一直以为SETNX是用来存数据的……尴尬）
参考: http://huoding.com/2015/09/14/463
### Jedis源码
在排查的过程中，顺便看了下Jedis的源码，这里也一并记录下来希望对人有帮助
```java
protected Connection sendCommand(final Command cmd, final byte[]... args) {
    try {
      connect();
      Protocol.sendCommand(outputStream, cmd, args);
      pipelinedCommands++;
      return this;
    } catch (JedisConnectionException ex) {
      /**发送有问题数据，会返回errorMessage*/
      try {
        String errorMessage = Protocol.readErrorLineIfPossible(inputStream);
        if (errorMessage != null && errorMessage.length() > 0) {
          ex = new JedisConnectionException(errorMessage, ex.getCause());
        }
      } catch (Exception e) {
      broken = true;
      throw ex;
    }
  }
```
可以看到jedis先把args塞到command中，然后把command放到Protocal的队列中
```java
private static void sendCommand(final RedisOutputStream os, final byte[] command,
      final byte[]... args) {
    try {
      os.write(ASTERISK_BYTE);
      os.writeIntCrLf(args.length + 1);
      os.write(DOLLAR_BYTE);
      os.writeIntCrLf(command.length);
      os.write(command);
      os.writeCrLf(); // -->作用是flushBuffer,立刻将缓存的数据写入预期目标

      for (final byte[] arg : args) {
        os.write(DOLLAR_BYTE);
        os.writeIntCrLf(arg.length);
        os.write(arg);
        os.writeCrLf(); // -->作用是flushBuffer
      }
    } catch (IOException e) {
      throw new JedisConnectionException(e);
    }
  }
```