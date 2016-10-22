title: OneToOneQueue —— 终极性能的无锁队列算法
date: 2016-03-26 16:37:41
categories: 多线程
tags: [Java,多线程,并发]
---

# 基本的队列设计

```Java
public class OneToOneConcurrentArrayQueue<E> implements Queue<E>{

	private final E[] buffer;

	private volatile long tail = 0;
	private volatile long head = 0;

	public OneToOneConcurrentArrayQueue(int capacity) {
	    buffer = (E[])new Object[capacity];
	}

	public boolean offer(E e) {
	    final long currentTail = tail;
	    final long wrapPoint = currentTail - buffer.length;  // tail位置往前移动length个位置
	    if(head <= wrapPoint) {
	        return false;
	    }
	    buffer[(int)(currentTail % buffer.length)] = e;
	    tail = currentTail + 1;
	    return true;
	}


	public E poll() {
	    final long currentHead = head;
	    if(currentHead >= tail) {
	        return null;
	    }
	    final int index = (int)(currentHead % buffer.length);
	    final E e = buffer[index];
	    buffer[index] = null;
	    head = currentHead + 1;
	    return e;
	}
}
```
这样的队列性能损耗主要是一下两点：  
1. 取余的计算  
2. volatile write以及该过程中可能发生的false sharing
因为我们探讨的OneToOne队列，那么显然volatile的刷新要求并不是很严格，可以不用每次的写操作都从cache中刷回主存  


---

# 改进 - 1

```Java
public final class OneToOneConcurrentArrayQueue2<E>
    implements Queue<E>
{
    private final int mask;
    private final E[] buffer;

    private final AtomicLong tail = new AtomicLong(0);
    private final AtomicLong head = new AtomicLong(0);

	public OneToOneConcurrentArrayQueue2(int capacity) {
		// 确保capacity是2^n
        capacity = findNextPositivePowerOfTwo(capacity);
        mask = capacity - 1;
        buffer = (E[])new Object[capacity];
    }

    public boolean offer(final E e)
	{
	    final long currentTail = tail.get();
	    final long wrapPoint = currentTail - buffer.length;
	    if (head.get() <= wrapPoint) {
	        return false;
	    }
	    // 替换了： buffer[(int)(currentTail % buffer.length)] = e;
	    // 优化了&操作
		buffer[(int)currentTail & mask] = e; 
		tail.lazySet(currentTail + 1);
	    return true;
	}

	public E poll()
	{
	    final long currentHead = head.get();
	    if (currentHead >= tail.get())
	    {
	        return null;
	    }
		final int index = (int)currentHead & mask; 
		final E e = buffer[index];
		buffer[index] = null; 
		head.lazySet(currentHead + 1);
		return e; 
	}
}
```
### 优化点1
注意这里使用`(int)currentTail & (capacity - 1)`优化了`buffer[(int)(currentTail % capacity)] = e;`的取余操作。  
不过这样的优化存在一个问题就是capacity必须是2^n   

### 优化点2
head和tail用AtomicLong替代long  
使用了lazySet来避免了频繁的volatile write在维持内存可见性时的内存损耗  

---

# 改进 - 2

```Java
public final class OneToOneConcurrentArrayQueue3<E> implements Queue<E>
{
	private final int capacity; 
	private final int mask; 
	private final E[] buffer;

	private final AtomicLong tail = new PaddedAtomicLong(0); 
	private final AtomicLong head = new PaddedAtomicLong(0);
	public static class PaddedLong {
		public long value = 0, p1, p2, p3, p4, p5, p6; 
	}
	private final PaddedLong tailCache = new PaddedLong(); 
	private final PaddedLong headCache = new PaddedLong();

	public boolean offer(final E e) {
		final long currentTail = tail.get();
		final long wrapPoint = currentTail - capacity; 
		if (headCache.value <= wrapPoint) {
			headCache.value = head.get();
			if (headCache.value <= wrapPoint) {
				return false; 
			}
		}
		buffer[(int)currentTail & mask] = e;
		tail.lazySet(currentTail + 1); 
		return true;
	}

	public E poll()
	{
		final long currentHead = head.get(); 
		if (currentHead >= tailCache.value) {
			tailCache.value = tail.get();
			if (currentHead >= tailCache.value) {
		        return null;
		    }
		}
		final int index = (int)currentHead & mask; 
		final E e = buffer[index];
		buffer[index] = null; 
		head.lazySet(currentHead + 1);
		return e; 
	}
}
```

### 优化点：
使用tailCache和headCache来避免对head和tail两个AtomicLong的频繁get()，读取普通变量肯定会比Atomic快很多  


# 实验结果
贴上参考文章中的实验结果：  


|Qps/Sec(Millions)|Mean Latency(ns)
---|---|---
LinkedBlockingQueue|4.3|~32000 / ~500
ArrayBlockingQueue|3.5|~32000 / ~600
ConcurrentLinkedQueue|13|NA / ~180
ConcurrentArrayQueue|13|NA / ~150
ConcurrentArrayQueue2|45|NA / ~120
ConcurrentArrayQueue3|150|NA / ~100


> Note: None of these tests are run with thread affinity set, Sandy Bridge 2.4 GHz Latency: Blocking - put() & take() / Non-Blocking - offer() & poll()


## 参考
[1. 终极性能的无锁队列算法](http://www.infoq.com/presentations/Lock-Free-Algorithms)

