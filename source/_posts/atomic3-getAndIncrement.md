title: 【不简单的Atomic工具】getAndAddInt in getAndIncrement
date: 2016-04-02 11:44:53
categories: atomic
tags: [Java,多线程,并发]
---

上一篇Atmoic文章中提到，JDK8在`AtomicInteger`的`getAndIncrement`方法实现上发生了变化：  

##### JDK7
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

##### JDK8
```Java
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
```

`getAndAddInt`
```Java
public final int getAndAddInt(Object paramObject, long paramLong, int paramInt) {
    int i;
    do
      i = getIntVolatile(paramObject, paramLong);
    while (!(compareAndSwapInt(paramObject, paramLong, i, i + paramInt)));
    return i;
}
```

```
jint
sun::misc::Unsafe::getIntVolatile (jobject obj, jlong offset)
{
  volatile jint *addr = (jint *) ((char *) obj + offset);
  jint result = *addr;
  read_barrier ();
  return result;
}
```

## 汇编之后
```
0x0000000002c207f5: lock cmpxchg %r8d,0xc(%rdx)
0x0000000002c207fb: sete   %r11b
0x0000000002c207ff: movzbl %r11b,%r11d
```

```
0x0000000002cf49c7: mov    %rbp,0x10(%rsp)
0x0000000002cf49cc: mov    $0x1,%eax
0x0000000002cf49d1: lock xadd %eax,0xc(%rdx)
```

原本的`lock cmpxchg`被`lock xadd`所替换，也就是在1.8之后，在硬件支持`lock XADD`这样的操作情况下，都会用fetch-and-add来替换CAS操作。



## 参考：
* [unsafe中getAndAddInt性能疑问](http://javagoo.tk/java/unsafe_getAndAddInt.html?utm_source=tuicool&utm_medium=referral)
