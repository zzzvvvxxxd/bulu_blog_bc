title: False Sharing问题
date: 2016-03-25 14:37:11
categories: 多线程
tags: [Java,多线程,并发]
---

## 背景

### Cache
对着多核的发展，CPU cache分成了三个级别：L1、L2、L3。L1是最接近CPU的cache，容量也是最小的，但是速度最快。  

从CPU到|大约需要的CPU周期|大约需要的时间（单位ns）
---|---|---|---
寄存器|1 cycle|NA
L1 Cache| 约3~4 cycles | 约0.5~1ns
L2 Cache| 约10~20 cycles | 约3~7ns
L3 Cache| 约40~45 cycles | 约15ns
跨槽传输 |    空            | 约20ns
内存    | 约120~240 cycles | 约60-120ns


### Cache line
为了高效的存取缓存，不是简单地随意地将单条数据写入缓存，缓存是由Cache line构成存储的最小单位，intel平台一般是64字节  
可以通过命令  
```Shell
$ cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size
```
Java中long占8个字节，再加上对象头（2字节）或者数组对象头（3字节），则可以获取7个long

### Cache条目
cache entry： 包含了如下部分
* cache line: 从主存一次copy的数据大小
* tag：标记cache line对应的主存的地址
* falg：标记当前cache是否invalid，如果是数据cache，还有是否dirty

### CPU的cache策略
1. cpu从不直接访问主存，通过cache间接访问
2. 每次需要访问主存时，遍历一遍全部cache line，查找主存的地址是否在某个cache line中
3. 如果cache中没有找到，则分配一个新的cache entry，把主存copy到cache line中，再从cache line中读取
同时，Cache中包含的cache entry有限，所以必须要有合理cache淘汰策略，一般是LRU，这里不再赘述。

## False Sharing
在多处理器，多线程的情况下，如果两个线程分别运行在不同CPU上，而其中某个线程修改了cache line中的元素，由于cache一致性原因，另一个线程的cache的line被宣告无效，在下一次访问时会出现一次cache line miss。  

如下图所示，thread1修改了memory灰化区域的第[2]个元素，而Thread0只需要读取灰化区域的第[1]个元素，由于这段memory被载入了各自CPU的硬件cache中，虽然在memory的角度这两种的访问时隔离的，但是由于错误的紧凑地放在了一起，而导致了，thread1的修改，在cache一致性的要求下，宣告了运行Thread0的CPU0的cache line非法，从而出现了一次miss，导致从小从memory中读取到cache line中，而一致性的代价付出了，但结果并不是thread0所care的，导致了效率的降低  

![figure 1](https://software.intel.com/sites/default/files/m/d/4/1/d/8/5-4-figure-1.gif)

> 在做多线程程序的时候,为了避免使用锁,我们通常会采用这样的数据结构:根据线程的数目,安排一个数组, 每个线程一个项,互相不冲突. 从逻辑上看这样的设计无懈可击,但是实践的过程我们会发现这样并没有提高速度. 问题在于cpu的cache line. 我们在读主存的时候,数据同时被读到L1,L2中去,而且在L1中是以cache line(通常64)字节为单位的. 每个Core都有自己的L1,L2,所以每个线程在读取自己的项的时候, 也把别人的项读进去, 所以在更新的时候,为了保持数据的一致性, core之间cache要进行同步, 这个会导致严重的性能问题. 这就是所谓的False sharing问题。

## 解决
打算分成两篇来写，嘿嘿

## 参考
[False Sharing问题](http://blog.csdn.net/pennyliang/article/details/5766541)  
[由一道淘宝面试题到False sharing问题](http://blog.csdn.net/wdzxl198/article/details/11218765)