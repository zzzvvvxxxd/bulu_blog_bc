title: 【不简单的Atomic工具】CAS
date: 2016-03-27 21:12:33
categories: atomic
tags: [Java,多线程,并发]
---

## CAS算法：
包含三个参数
```
CAS(V, E, N)
```
* V 表示要更新的变量
* E 表示预期值
* N 表示新值

仅当V值等于E值时，才会将V的值设置为N。如果V的值和E不同，表示已经有其他线程对其进行了更新，则当前线程什么都不做。最后CAS返回V的真实值。

CAS是抱着乐观的态度进行的，总认为自己可以成功的完成操作。当多个线程同时使用CAS操作一个变量时，只有一个会胜出。失败的线程不会被挂起，而是返回失败信息，并允许再次尝试。

CPU指令：
```asm
cmpxchg
# accumulator = AL, AX, or EAX, depending on whether a byte, word, or doubleword comparison is being performed
```
该指令的基本逻辑如下：
```Java
if(accumulator == Destination) {
    ZF = 1;
    Destination = Source;
} else {
    ZF = 0;
    accumulator = Destination;
}
```

## 缺点：
1. ABA问题。
> 因为 CAS 需要在操作值的时候检查下值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。ABA问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A－B－A 就会变成1A-2B－3A。
从Java1.5开始JDK的atomic包里提供了一个类 AtomicStampedReference 来解决 ABA 问题。这个类的compareAndSet 方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。
```Java
public boolean compareAndSet
        (V      expectedReference,//预期引用
         V      newReference,//更新后的引用
        int    expectedStamp, //预期标志
        int    newStamp) //更新后的标志
```

2. 循环时间长开销大。
> 自旋 CAS 如果长时间不成功，会给 CPU 带来非常大的执行开销。如果JVM能支持处理器提供的 pause 指令那么效率会有一定的提升，pause 指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使 CPU 不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。第二它可以避免在退出循环的时候因内存顺序冲突（memory order violation：内存顺序冲突一般是由伪/假共享引起，假共享是指多个 CPU 同时修改同一个缓存行的不同部分而引起其中一个CPU的操作无效，当出现这个内存顺序冲突时，CPU必须清空流水线）而引起 CPU 流水线被清空（CPU pipeline flush），从而提高 CPU 的执行效率。

3. 只能保证一个共享变量的原子操作。
> 当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁，或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如有两个共享变量i＝2,j=a，合并一下ij=2a，然后用CAS来操作ij。从Java1.5开始JDK提供了AtomicReference类来保证引用对象之间的原子性，你可以把多个变量放在一个对象里来进行CAS操作。

## AtmoicInteger中的CAS实现

**JDK7**:
```Java
public final int getAndIncrement() {
  for (;;) {
    int current = get();
    int next = current + 1;
    if (compareAndSet(current, next))
      return current;
  }
}
```

**JDK8**:
```Java
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
```
可以看到JDK8在`getAndIncrement()`方法的实现上做出了一定的改变，在[AtomicInteger Java 7 vs Java 8](http://ashkrit.blogspot.com/2014/02/atomicinteger-java-7-vs-java-8.html)中列出了两个版本的benchmark，JDK8的实现大约提升了1.5~2倍的性能，作者也直呼`Unsafe`变得更加与有用了。
