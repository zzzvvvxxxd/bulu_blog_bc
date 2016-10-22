title: 【深入理解Java虚拟机2-笔记】可达性分析&引用
date: 2015-06-07 11:44:53
categories: JVM
tags: [Java,JVM]
---

### 引用计数法

Java并没有使用传统的引用计数法，理由很简单，该算法无法很好的解决对象间相互循环引用的问题。  

### 可达性分析算法
Java（甚至是C#和Lisp）的主流实现中，都是使用可达性分析（Reachability Chain）来判定对象是否存活的。
> 基本思路：通过一系列的成为`GC Roots`的对象作为起点，从这些起点开始向下搜索，搜索过程的路径称为`Reference Chain`，当一个对象没有任何引用链相连时，则证明此对象是不可用的。所以即便有些对象相互引用，但是因为和`GC Roots`之间不可达，依然会被GC掉。  

作为`GC Roots`的对象包括：
* 虚拟机栈（栈帧中的本地变量表）中引用的对象
* 方法区中类静态属性引用的对象
* 方法区中常量引用的对象
* 本地方法栈中JNI引用的对象

### 引用
无论是通过引用计数法还是可达性分析来判断对象是否存活，都和“引用”相关。
* 强引用（Strong Reference）
> 类似`Object obj = new Object();`这类引用，只要强引用还在，GC就不会回收

* 软引用（Soft Reference）
> 用来描述一些还有用，但并非必须的对象。对于软引用关联的对象，在OOM之前，将会把这些对象进行二次回收。如果之后还是没有足够内存，就会跑出内存溢出异常。

* 弱引用（Weak Reference）
> 同样用来描述非必须对象，但是强度低于弱引用。在下一次GC时一定会被回收。

* 虚引用（Phantom Reference）
> 一个对象是否有虚引用，完全不会对其生存时间造成影响，也无法通过虚引用获得一个对象实例。唯一目的就是可以在这个对象呗收集器回收时收到一个系统通知。  
一旦GC决定一个“obj”是虚可达的，它（指PhantomReference）将会被入队到对应的队列，但是它的指代并没有被清除。也就是说，虚引用的引用队列一定要明确地被一些应用程序代码所处理。  
虚引用在实现一个对象被回收之前必须做清理操作是很有用的。有时候，他们比`finalize()`方法更灵活。

补充：
1. 软引用可以用来实现Caching和pooling
2. 可以使用弱引用来实现对HashMap/HashSet中对象回收，但是要看使用场景。
```
class Registry {
     private Set registeredObjects = new HashSet();

     public void register(Object object) {
         registeredObjects.add( new WeakReference(object) ); //如果不适用WeakReference则HashSet中的对象永远不会回收
     }
 }
```
3. 虚引用可以用来替代`finalize()`做一些回收之前的操作。例如，page缓存在gc之前需要被刷回磁盘。

### 参考
[1. Java 如何有效地避免OOM：善于利用软引用和弱引用](http://www.cnblogs.com/dolphin0520/p/3784171.html)
