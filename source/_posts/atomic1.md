title: 【不简单的Atomic工具】lazySet & set
date: 2016-03-24 22:40:11
categories: atomic
tags: [Java,多线程,并发]
---

在研究Martin Thompson的演讲[终极性能的无锁队列算法](http://www.infoq.com/presentations/Lock-Free-Algorithms)时遇到了AtomicLong的lazySet方法，作者用这个方法替代了set。之前在公司设计MulSemaphore类时，我就并没有注意到Atomic类还有这样一个方法。  

## lazySet()
经常在维护一个非阻塞的数据结构时，内部显然常常会有volatile成员，而对于通信队列这样的对性能要求极端的数据结构，JMM要维持volatile的内存可见性，在对这样的成员写操作时，会有频繁的cache的读写（save和load？）到主内存。这样的时间耗损队形能而言，其实也十分重要。  
lazySet()可以让一个线程在没有其他线程volatile写或者synchronized动作发生前，本地缓存的写操作不必刷回主内存。  

```Java
public final void lazySet(int newValue) {
    unsafe.putOrderedInt(this, valueOffset, newValue);
}
```
lazySet的本质就是putOrderInt()，该函数是putIntVolatile()方法的延迟实现。  

正是因为这样的特性，使得lazySet适合于SPSC的数据结构中，对于单Producer单Consumer这样的结构，延迟写并不是什么问题。同时还优化了store-load屏障在性能上的损耗（store-store屏障替代）。 