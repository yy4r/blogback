---
title: IO 小计
date: 2018-07-15 21:43:14
tags: IO
toc: true
cover: http://a0.att.hudong.com/07/01/31300543122909143996016035883.jpg
---
我们都知道IO，网络，内存，CPU是影响计算机性能的比较重要的几个部分。对于I/O，我们经常仅仅停留于概念层的理解，这次我们不妨做点小事情来认识下I/O。来看下如何优化IO来达到提升机器性能。

> 给出一个题目： 将一个2GB左右的大型文件，复制到另一处位置，需要多少时间呢？

<!--more-->

如果你是一个Java开发者，那么可选的方案有以下三者，让我们动手写几个Demo来试一下。

1. java IO
2. java NIO
3. java MappedByteBuffer（内存映射）

#### JAVA IO 方案：

```java
public void readByIO() throws IOException {

        BufferedInputStream bufferedInputStream = null;
        BufferedOutputStream bufferedOutputStream = null;

        try {
            Integer bufferSize = 20 * 1024 * 1024;
            byte[] buffer = new byte[bufferSize];
            File file = new File(path);
            bufferedInputStream = new BufferedInputStream(new FileInputStream(file));
            bufferedOutputStream = new BufferedOutputStream(new FileOutputStream(new File(IO_OUT_FILE)));
            while (bufferedInputStream.read(buffer) > 0) {
                bufferedOutputStream.write(buffer);
            }

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            bufferedInputStream.close();
            bufferedOutputStream.close();
        }

    }
```

#### JAVA NIO 方案：

```java
FileChannel inputFileChannel = null;
        FileChannel outPutFileChannel = null;
        try {
            inputFileChannel = new FileInputStream(new File(path)).getChannel();
            outPutFileChannel = new FileOutputStream(new File(NIO_OUT_FILE)).getChannel();
            outPutFileChannel.transferFrom(inputFileChannel, 0, inputFileChannel.size());
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            inputFileChannel.close();
            outPutFileChannel.close();
        }
```

#### JAVA MappedByteBuffer 方案：

```java
public void readByMapping() throws IOException {
        Integer bufferSize = 20 * 1024 * 1024;
        RandomAccessFile randomAccessFile = new RandomAccessFile(new File(path), "r");
        FileChannel outPutFileChannel = new FileOutputStream(new File(MAPPING_NIO_OUT_FILE2)).getChannel();

        for (int i = 0; i * bufferSize < randomAccessFile.length(); i++) {
            long leave = bufferSize;
            if ((i + 1) * bufferSize > randomAccessFile.length()) {
                leave = randomAccessFile.length() - (i * bufferSize);
            }

            MappedByteBuffer mappedByteBuffer = randomAccessFile.getChannel().map(FileChannel.MapMode.READ_ONLY, i * bufferSize, leave);
            outPutFileChannel.write(mappedByteBuffer);
        }
    }
```
#### 结果
1. 方案一：10980ms
2. 方案二：14528ms
3. 方案三不使用buffer的情况：24851ms
4. 方案三：8023ms

可以看到，但你使用不同的方式来使用IO的时候，其实对于机器的性能还是有很大的影响的。那么如何才能够提升我们使用IO的效率呢？笔者不才，搜集了一些资料来和大家分享一下。

### 物理层IO性能相关
对于磁盘有两个衡量的指标IOPS以及吞吐量：

1. IOPS指的是（Input/Output Per Second）关注的随机读写能力
2. 吞吐量一般针对顺序读写能力；

#### IOPS概念
针对IOPS，我们可以通过一些指标来更清晰的理解这个概念，以7200转的机械硬盘为例，

IOPS = 1000 ms/ (寻道时间 + 旋转延迟)。可以忽略数据传输时间。
7200转/分的STAT硬盘平均物理寻道时间是9ms
7200  rpm的磁盘平均旋转延迟大约为 60*1000/7200/2 = 4.17ms

**rpm的磁盘IOPS = 1000 / (9 + 4.17)  = 76 IOPS**

#### 吞吐量概念
吞吐量比较简单就是顺序读的时候，磁盘所能输出的流量，例如128MB/s

#### RAID 磁盘阵列
当你无法提升单块磁盘性能的时候，这时就可以使用RAID 磁盘阵列。RAID分为0~5，针对不同的业务情况，可以选择不同的策略。一般来说，都是使用RAID0，然后通过在不同的服务器上做备份（例如HDFS）。
RAID0可以理解为有n块磁盘，n块磁盘同时写入，这样就能够大大提升写入性能。
