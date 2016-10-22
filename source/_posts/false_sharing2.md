title: False Sharing问题（2）
date: 2016-03-26 16:32:38
categories: 多线程
tags: [Java,多线程,并发]
---


## 如果避免频繁的false sharing造成的性能损耗：

**PaddedLong**  
```Java
public static class PaddedLong {
    public long value = 0, p1, p2, p3, p4, p5, p6; 
}
```

**PaddedAtomicLong**  
```Java
public class PaddedAtomicLong extends AtomicLong {
    public PaddedAtomicLong() {
    	super();
    }

    public PaddedAtomicLong(final long initialValue) {
        super(initialValue);
    }

    public volatile long p1, p2, p3, p4, p5, p6 = 7;
}
```

## 为何是插入7个long
一般而言cache line是64byte，而每个Java对象都有一个2word（2*4byte）的对象头，而long本身是8byte。  

## PaddedLong的应用
一般是用在频繁读写的情况下，可以参考之后的OneToOne队列的优化文章