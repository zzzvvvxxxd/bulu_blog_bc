title: 【Java多线程-1】中断线程
date: 2015-06-10 21:12:32
categories: blog
tags: [Java,多线程,并发]
---

# 中断
```Java
Thread.interrupt()
```  
当一个线程执行时，另一个线程可以调用Thread对象的interrupt()方法来中断它



## interrupt()方法
在目标线程中设置一个标志，表示它已经被中断。类似sleep这样的方法，会检查该标志位，如果被置位会抛出InterruptedException异常  
目标线程可以catch到InterruptedException，但是并不会中断该线程try-catch之后的代码执行，如果catch语句块中没有类似return的语句，则会在catch之后继续顺序执行。  



## isInterrupted()方法
判断中断状态
可以在Thread对象上调用isInterrupted()方法来检查任何线程的中断状态。
## 注意：
线程一旦被中断，isInterrupted（）方法便会返回true，而一旦sleep()方法抛出异常，它将清空中断标志，此时isInterrupted()方法将返回false。

```Java
// 中断该线程
// 除了是线程自己interrupt自己，否则都要使用checkAccess函数进行权限检查
// 可能会引起SecurityException()
// 如果线程被wait() join() sleep()函数阻塞住，那么其interrupt的状态会被清空
// 并且会抛出InterruptedException
// 如果线程被阻塞在InterruptibleChannel上的nio操作，那么该channel会被关闭
// 线程的interrupt状态会被置位，线程也会收到ClosedByInterruptException异常
// 如果线程在nio.channels.Selector上阻塞，则线程的interrupt状态会被置位
// 并且会从选定的操作中立即返回...
// 如果不是上述的情况，那么线程的interrupt状态将被设置
// 对非alive的线程，使用该方法，没有任何作用
public void interrupt() {
    if (this != Thread.currentThread())
        checkAccess();
    synchronized (blockerLock) {
        Interruptible b = blocker;
        if (b != null) {
            interrupt0();           // Just to set the interrupt flag
            b.interrupt(this);
            return;
        }
    }
    interrupt0();
}
```  

## Thread.interrupted()方法判断中断状态
可以使用Thread.interrupted()方法来检查当前线程的中断状态（并隐式重置为false）。又由于它是静态方法，因此不能在特定的线程上使用，而只能报告调用它的线程的中断状态，如果线程被中断，而且中断状态尚不清楚，那么，这个方法返回true。   
与isInterrupted()不同，它将自动重置中断状态为false，第二次调用Thread.interrupted()方法，总是返回false，除非中断了线程。  
建议使用isInterrupted()方法   


## 示例代码：

```Java
public class ThreadTest implements Runnable{
 	  public void run() {
		      try{
			         System.out.println("in run() - sleep about 20s ...");
			         Thread.sleep(20000);
        		 	System.out.println("in run() - wake up!");
		      } catch (InterruptedException e) {
			         System.out.println("in run() - interrupted while sleeping!");
			         return;		//返回run调用处
		      }
		      System.out.println("in run() - leaving normally");
	   }
	
	   public static void main(String[] args) throws Exception{
		      ThreadTest r = new ThreadTest();
		      Thread t = new Thread(r);
		      t.start();
		      Thread.sleep(2000);
		      System.out.println("in main() - interrupting!");
		      t.interrupt();
		      System.out.println("in main() - leaving");
	   }
// END_OF_CLASS
}
```  
输出：  
run()方法sleep大于2s之后，被中断，并且立即返回  
同样是2s被终端了try中的操作，但是继续顺序执行之后的代码，并不是立即返回   

## 待决中断    
sleep()方法的实现检查到休眠线程被中断，它会相当友好地终止线程，并抛出InterruptedException异常。另外一种情况，如果线程在调用sleep()方法前被中断，那么该中断称为**【待决中断】**    
它会在遇到sleep())方法时，立即抛出InterruptedException异常。    



> 我目前的理解就是，Thread.interrupt()只是改变了Thread内部的中断标识，不会终止程序的运行
当运行到sleep方法时，会检查中断标识，并抛出异常，然后就是用户处理的部分了















