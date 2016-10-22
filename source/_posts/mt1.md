title: 【Java多线程-0】综述
date: 2015-05-29 07:56:29
categories: 多线程系列
tags: [Java,多线程,并发]
---

# 综述 

## 并发编程的挑战：  

1. 上下文切换   
    可能出现并行程序性能不能最优化，甚至慢与串行程序  
2. 死锁  
3. 资源限制的挑战  
    受限与资源的限制，可能将并行程序最终按照了串行方式执行（例如，某一资源同时只允许一个线程访问）  

 
线程：  
   程序中单独顺序的控制流，本身依靠程序进行运行  
进程：  
   执行中的程序  



## 线程状态
创建状态：new
就绪状态：调用了start()等待调度
运行状态：执行run()
阻塞状态：暂停执行，可能在等待IO
终止状态：线程销毁



## 线程的优先级

1 :  MIN_PRIORITY    
10:  MAX_PRIORITY  
5 :  NORM_PRIORITY(默认)    

通过setPriority()方法设置  
  
## 实现方式 
两种实现线程的方式   


1. 继承Thread类   
2. 实现Runnable接口    


```Java
class ClassName extends Thread{
    Run()
}  
```
线程调度器来分别调用所有线程的run()方法   


## Thread 常用方法 

```Java
new Thread()	//创建
new Thread(String name)
new Thread(Runnable target)
new Thread(
    Runnable target,
    String name
)

void start()	//启动线程

static void sleep(long millis)	//线程休眠
static void sleep(
    long millis,
    int nanos
)

void join()	//使其他线程等待当前线程终止
void join(long millis)
void join(
    long millis,
    int nanos
)

stativ void yield()	//当前运行线程释放处理器资源
//获取线程引用	
static Thread currentThread()	//返回当前运行的线程引用
//是否启动
boolean isAlive()	//是否启动
//优先级	
void setPriority(int newPriority)	//设置线程优先级
```  

## 继承Thread和实现Runnable接口的区别	  
#### (1)适合多个相同程序代码的线程去处理同一资源的情况      


把虚拟CPU（线程）同程序的代码，数据有效的分离，较好地体现了面向对象的设计思想。   



#### (2)可以避免由于Java的单继承特性带来的局限。  



我们经常碰到这样一种情况，即当我们要将已经继承了某一个类的子类放入多线程中，由于一个类不能同时有两个父类，所以不能用继承Thread类的方式，那么，这个类就只能采用实现Runnable接口的方式了。     


#### (3)有利于程序的健壮性，代码能够被多个线程共享，代码与数据是独立的。  


当多个线程的执行代码来自同一个类的实例时，即称它们共享相同的代码。多个线程操作相同的数据，与它们的代码无关。当共享访问相同的对象是，即它们共享相同的数据。当线程被构造时，需要的代码和数据通过一个对象作为构造函数实参传递进去，这个对象就是一个实现了Runnable接口的类的实例。    