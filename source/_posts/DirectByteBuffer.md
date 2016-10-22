title: 【天池比赛日记 - 2】MaxDirectMemorySize和堆外内存
date: 2016-07-22 21:03:11
categories: JVM
tags: [Java,JVM]

---

最近在做阿里的中间件比赛，在第二题的背景下，我们的方案需要一个基于磁盘的map和B+索引，显然IO就成为了瓶颈之一。为了提升IO的速度，我们选择`MappedByteBuffer`和`channel.map()`作为我们的IO处理方案。但是这样的方案存在一个问题，堆外内存不受JVM的控制，可能会因为一些原因导致JVM Crash，所以对这个问题做一点必要的研究。

## 1. 首先

首先，根据JDK中对`MappedByteBuffer`的说明：  
> A direct byte buffer whose content is a memory-mapped region of a file.
Mapped byte buffers are created via the FileChannel.map method. This class extends the ByteBuffer class with operations that are specific to memory-mapped file regions.  

`MappedByteBuffer`本质上就一种继承了`ByteBuffer`的direct byte buffer，是分配在JVM堆外的内存。因而也就不受JVM的控制，也只有在full gc的时候才会被回收。  
虽然，堆外内存包含了：
* jvm本身在运行过程中分配的内存
* codecache
* jni里分配的内存
* DirectByteBuffer分配的内存

我们在比赛过程中更关注的是DirectByteBuffer所产生的堆外内存，包括它的配置和回收等操作。  

## 2. 然后
【比赛陷入僵局，结束了再续写】
