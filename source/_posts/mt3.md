title: 【Java多线程-2】线程挂起、恢复、终止 
date: 2015-06-11 11:29:08
categories: blog
tags: [Java,多线程,并发]
---

类似suspend()方法的挂起方式，已经被java丢弃, 较为合理的方式是设置挂起的标识位   

首先在Runnable对象中设置标志位suspended  


```Java  
private volatile boolean suspended;  
```  


volatile变量告诉JVM，这个变量将要被外部的线程修改，具体的参考下一篇笔记吧   
然后设置置位和消除suspended状态方法   


```Java
public void suspendRequest() {  
    suspended = true;  
}  
  
public void resumeRequest() {  
    suspended = false;  
}  
```  


还有一个挂起时用于等待的方法：   


```Java
private void waitWhileSuspended() throws InterruptedException {  
    //这是一个“繁忙等待”技术的示例。  
    //它是非等待条件改变的最佳途径，因为它会不断请求处理器周期地执行检查，   
    //更佳的技术是：使用Java的内置“通知-等待”机制  
    while ( suspended ) {  
        Thread.sleep(200);  
    }  
}  
```   


接下来在Runnable对象的run()方法所要保护起来的代码段前后添加waitWhileSuspended方法来确保进入和退出代码块的时候，都可以检测到suspend信号，从而可以挂起


```Java  
public void run() {
    while(true) {
        waitWhileSuspended(); 
  
        // do sth ...
        waitWhileSuspended();
    }
}
```  


而产生suspend信号的线程只需要在需要线程挂起的时候调用suspendRequest()方法,解除suspend怎使用resumeRequest()方法而Runnable对象中被waitWhileSuspended();包围的代码段不会被suspend信号终止

# 终止
在Thread的run()方法中return，线程会自然消亡  

如果使用stop()线程突然终止，很少有机会进行清理工作会释放当前所持有的锁，而锁的突然解除，会对数据一致性产生危险，甚至程序崩溃解决方案还是使用标志位,然后在程序中使用while来检测标志位