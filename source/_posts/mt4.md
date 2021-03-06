title: 【Java多线程-4】volatile
date: 2015-06-14 09:13:21
categories: blog
tags: [Java,多线程,并发]
---

# volatile写-读的内存语义

## 概念

概念|描述
---|---|---
缓存行	cache line	| 缓存中可以分配的最小存储单元，处理器填写缓存线时，回家在整个缓存线，需要使用多个内存读周期
缓存行填充	cache line fill| 当处理器识别到从内存中读取操作是可缓存的，处理器读取整个缓存行到适当的缓存
缓存命中	cache hit|	如果进行告诉缓存行填充操作的内存位置仍是下次处理器访问的位置，处理器从缓存中读取操作数，而不是内存

```Java
instance = new Singleton();
```  

其对应的汇编代码为：  




```asm
0x01a3de1d: movb $0x0, 0x1104800(%esi);
0x01a3de24: lock addl $0x0, (%eps);
```  

**lock前缀的指令在多核处理器会引发：**将当前处理器缓存行的数据写回到系统内存, 这个写回内存的操作会使其他CPU中缓存了该内存地址的数据无效
为了优化速度，处理器不直接和内存通信，一般是将系统内存数据读取到本地内存（L1，L2或其他），但是操作并不保证何时写回到主存。如果是volatile变量，在写入时，回向处理器发送Lock指令这个变量所在的缓存行的数据会写回到系统内存。这里就有一个缓存一致性协议，每个处理器嗅探在总线上传播的数据来检查自己的缓存值是否过期。如果处理器发现自己缓存行对应的内存地址被修改，就会将当前的处理器缓存设置为无效状态。该处理器再一次对该缓存数据的操作，就会先从主存中读取。  

## volatile实现原则  

* Lock前缀指令会引发处理器缓存写回到内存  
* 一个处理器的缓存回写到内存会使得其他处理的缓存无效  

也就是：

> 当写一个volatile变量时，JMM会把该线程对应的本地内存中（本地内存就是上文中的当前处理器缓存）的共享变量（实际应当是volatile变量所在的缓存行）刷新到主内存。所以，很多时候一些非volatile变量的写操作发生在volatile变量之前修改，因为这里的刷新机制，所以也无意中实现了可见性（很多blog中的代码都有这个问题...所以没有拿到理想的输出，例如：http://blog.csdn.net/ns_code/article/details/17101369）  


---


# 简述
volatile是Java提供的一种弱同步机制  

在Java1.2之前，Java内存模型总是从主存（即共享内存）中读取变量，也就是不同线程读取的是同一块内存。而后来随着JVM优化，在多线程环境下volatile关键字的作用越来越明显。    
当前的JMM模型下，线程可以把变量保存在本地内存，而不是直接在主存中读写。  
后果就是：  
> 一个线程在主存中修改了变量的值,另一个线程还在使用其拷贝到寄存器中的值,数据就产生了不一致性.    



volatile就指示JVM改变量是不稳定的，每次操作都要在内存读取。在多线程环境下，任何多任务共享的变量，都应该是volatile的。新值可以立即同步到主存（即共享内存）。 每次使用前，从主存刷新。在任何时刻，两个不同的线程总是看到某个成员变量的同一个值在任何时刻，两个不同的线程总是看到某个成员变量的同一个值。 说白了就是：**禁止线程私有拷贝**  

volatile是一种稍弱的同步机制，在访问volatile变量时不会执行加锁操作，也就不会执行线程阻塞，因此volatile变量是一种比synchronized关键字更轻量级的同步机制



# 综合来说：

* volatile 变量对所有线程是立即可见的,对 volatile 变量所有的写操作都能立即反应到其他线程之中 
* 也就是说volatile变量在各个线程中是一致的                                               
* 但是基于volatile的运算仍旧不算是线程安全的      

> 在遇到多个线程共享的变量时都加上volatile，但是不要将其看成是线程安全的
