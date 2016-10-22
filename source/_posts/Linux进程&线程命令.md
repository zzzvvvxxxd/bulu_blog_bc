title: 【Linux命令】照看好咱们的进程和线程
date: 2016-08-20 17:36:53
categories: Linux
tags: [Linux]
---
### 1. 查找占CPU最多的线程
步骤1： 查找出占用CPU最高的java线程pid
```Shell
top -Hp 18207   #查看运行进程18207内所有线程的CPU消耗情况
```
可以找到`%CPU`项最高的一个pid，就是该线程，假设pid是18250，将其转换为16进制：0x474a    

步骤2： jstack查看各个线程栈
```Shell
jstack 18207
```
找到nid为0x474a的线程栈信息，其中包含了该线程的name，OK定位到了   

### 2. 查看一个进程运行了多久

```
ps -p {PID-HERE} -o etime
ps -p {PID-HERE} -o etimes
ps -p {PID-HERE} -o etime=
ps -p {PID-HERE} -o etimes=
```

实际上可以打印的信息有很多：
```
ps -p 6176 -o pid,cmd,etime,uid,gid
```

### 3. 统计一个进程的线程数
方法1： `ps`
```
ps hH p <pid> | wc -l
```
方法2：`/proc`
proc 伪文件系统，它驻留在 /proc 目录，这是最简单的方法来查看任何活动进程的线程数。 /proc 目录以可读文本文件形式输出，提供现有进程和系统硬件相关的信息如 CPU、中断、内存、磁盘等等  
```
cat /proc/<pid>/status
```
上面的命令将显示进程 <pid> 的详细信息，包括过程状态（例如, sleeping, running)，父进程 PID，UID，GID，使用的文件描述符的数量，以及上下文切换的数量


### 4. 查看某个进程的线程
方法1：
```
ps -T -p <pid>
```
> `SID`栏表示线程ID，而`CMD`栏则显示了线程名称

方法2：
`top`可以使用`-H`开启线程查看模式，如果要查看特定进程的线程：
```
top -H -p <pid>
```
方法3：
要在`htop`中启用线程查看，请开启`htop`，然后按`<F2>`来进入`htop`的设置菜单。选择“设置”栏下面的“显示选项”，然后开启`树状视图`和`显示自定义线程名`选项。按`<F10>`退出设置  
Htop的具体使用后续会专门写一篇记录一下。  

### 5. 查看进程被运行在哪个CPU上
方法1：`taskset`  
```
taskset -c -p <pid>
```
方法2：`ps`
```
ps -o pid,psr,comm -p pid
```
方法3：`top`
```
top -p <pid> # 好处是可以持续观察
```
