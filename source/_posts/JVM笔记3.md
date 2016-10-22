title: 【深入理解Java虚拟机3-笔记】GC算法和垃圾收集器
date: 2015-06-12 11:44:53
categories: JVM
tags: [Java,JVM]
---
# 垃圾回收算法
## 1. 标记回收算法

* 标记
* 回收

> 标记出所有需要回收的对象，在标记完成后  统一回收所有标记的对象

不足：
1. 效率问题。标记和清楚两个过程效率都不高
2. 空间问题。会产生大量内存碎片，导致后续要分配较大对象时，无法找到足够的内存而不得不提前触发垃圾收集动作。

## 2. 复制算法 Copying
为了解决效率问题，复制算法将可用内存按容量划分为大小相同的两块，每次只使用其中一块，当这一块内存用完，就将活着的对象复制到另一块上，然后再把已使用的内存空间一次清理掉。不需要考虑内存碎片的情况，只需要移动堆顶指针，按顺序分配内存即可，**实现简单，运行高效**。  

* Eden  
* Survivor0  
* Survivor1  

HotSpot中默认的Eden和Survivor的比例是8:1，也就是每次新生代中可用内存空间为整个新生代容量的90%。当Survivor空间不够时，需要依赖其他内存（老年代）进行分配担保（Handle Promotion）。  

## 3. 标记-压缩算法 Mark-Compact
适用于老年代。  
标记过程和标记-清楚算法一致，后续让所有存活对象都向一端移动，然后直接清理掉端边界意外的内存。

# 垃圾收集器
![总览](https://raw.githubusercontent.com/zzzvvvxxxd/Picture/master/hexo/gc-all.png)  

上图展示了7种作用于不同分代的收集器，如果两个收集器之间存在连线，就说明可以搭配使用

## Serial收集器
最基本、发展历史最悠久的收集器。是一个单线程的收集器，它只会使用一个CPU或一条线程来完成垃圾收集的工作，同时还必须暂停其他的所有工作线程，知道收集结束。比较适合client模式下的客户端或者单核的场景。
![Serial收集器](https://raw.githubusercontent.com/zzzvvvxxxd/Picture/master/hexo/serial.png)  
## ParNew收集器
本质就是Serial收集器的多线程版本
![ParNew收集器](https://raw.githubusercontent.com/zzzvvvxxxd/Picture/master/hexo/parnew.png)
目前只有Serial和ParNew可以和老年代的CMS收集器配合工作，而后者目前是最多使用的收集器。自然ParNew也成了Server模式下首选的新生代收集器。  
ParNew是使用`-XX:+UseConcMarkSweepGC`的默认新生代收集器，也可以通过`-XX:+UseParNewGC`来明确指定它。    
如果要设置ParNew的线程数，可以使用`-XX:ParallelGCThreads`来限制。  

## Parallel Scavenge收集器
使用复制算法的新生代收集器。CMS等收集器的关注点是尽可能缩短垃圾收集时用户线程的停顿，而Parallel Scavenge收集器的目的则是达到一个可控制的吞吐量（Throughput）。前者可以有更好的响应时间和用户体验，而后者则可以高效率地利用CPU，尽快完成计算密集型任务。  
> 吞吐量是CPU用于运行用户代码的时间与CPU总消耗时间的比值。  
吞吐量 = 运行用户代码时间 / (运行用户代码时间 + 垃圾收集时间)

相关参数：
```
-XX:MaxGCPauseMillis  # 设置最大垃圾收集时间
-XX:GCTimeRatio   # 直接设置吞吐量，默认值为99，即为允许最多1%的GC时间
-XX:+UseAdaptiveSizePolicy  # Parallel Scavenge支持自适应调节 P80
```

## Serial Old收集器
Serial收集器的老年代版本，使用标记-整理算法  

## Parallel Old收集器
Parallel Scavenge的老年代版本，使用多线程标记-整理算法

## CMS
![CMS收集器](https://raw.githubusercontent.com/zzzvvvxxxd/Picture/master/hexo/cms.png)
是一个以获取最短停顿时间为目标的收集器，基于`标记-清除`算法。  
* 初始标记（CMS initial mark）
> 仅仅标记一下GC Roots能直接关联到的对象

* 并发标记（CMS concurrent mark） --> stop the world
> 进行GC Roots Tracing过程

* 重新标记（CMS remark）
> 修正并发标记期间因用户程序继续运作导致标记产生变动的那一部分对象的标记记录，停顿时间稍长于初始标记，远小于并发标记

* 并发清除（CMS concurrent sweep） --> stop the world

明显缺点：  
###### CMS收集器对CPU资源非常敏感

> 其默认启动的收集线程数=(CPU数量+3)/4，在用户程序本来CPU负荷已经比较高的情况下，如果还要分出CPU资源用来运行垃圾收集器线程，会使得CPU负载加重。

###### CMS无法处理浮动垃圾(Floating Garbage)，可能会导致`Concurrent ModeFailure`失败而导致另一次Full GC

> 由于CMS收集器和用户线程并发运行，因此在收集过程中不断有新的垃圾产生，这些垃圾出现在标记过程之后，CMS无法在本次收集中处理掉它们，只好等待下一次GC时再将其清理掉，这些垃圾就称为浮动垃圾。
CMS垃圾收集器不能像其他垃圾收集器那样等待年老代机会完全被填满之后再进行收集，需要预留一部分空间供并发收集时的使用，可以通过参数`-XX:CMSInitiatingOccupancyFraction`来设置年老代空间达到多少的百分比时触发CMS进行垃圾收集，默认是68%。
如果在CMS运行期间，预留的内存无法满足程序需要，就会出现一次ConcurrentMode Failure失败，此时虚拟机将启动预备方案，使用Serial Old收集器重新进行年老代垃圾回收。

###### CMS收集器是基于标记-清除算法，因此不可避免会产生大量不连续的内存碎片

> 如果无法找到一块足够大的连续内存存放对象时，将会触发因此Full GC。CMS提供一个开关参数`-XX:+UseCMSCompactAtFullCollection`，用于指定在Full GC之后进行内存整理，内存整理会使得垃圾收集停顿时间变长，CMS提供了另外一个参数`-XX:CMSFullGCsBeforeCompaction`，用于设置在执行多少次不压缩的Full GC之后，跟着再来一次内存整理。
