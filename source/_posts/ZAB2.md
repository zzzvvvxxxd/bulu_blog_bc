title: 【一致性协议05】 ZAB协议详细介绍
date: 2016-08-22 01:26:47
categories: 分布式
tags: [分布式,一致性协议]
---

## 一、消息广播
### 1.1 类似二阶段提交的消息广播
ZAB协议的消息广播过程本质就是一个简单的原子广播协议，类似于`2PC`。针对客户端的事务请求，Leader服务器会为其是生成对应的事务`Proposal`，并将其发送给集群中其余的所有机器，然后再分别收集各自的选票，最后进行事务提交。  
![ZAB协议消息广播流程示意图](https://raw.githubusercontent.com/zzzvvvxxxd/Picture/master/hexo/ZAB%E6%B6%88%E6%81%AF%E5%B9%BF%E6%92%AD.JPG)

和`2PC`略有不同，这里的二阶段提交过程移除了中断（abort）逻辑。所有的Follower要么正常反馈，要么就抛弃Leader服务器。同时ZAB协议将二阶段提交中的中断逻辑移除意味着我们可以在过半的Follower服务器已经反馈`Ack`之后就开始提交`Proposal`了，而不需要等待集群中的所有Follower服务器都反馈响应。    

在这种简化的二阶段提交模型下，无法处理Leader服务器崩溃带来的数据不一致问题，因此引入了**崩溃回复模式**。  

### 1.2 ZXID
Leader会为每个事物Proposal分配一个全局唯一的单调递增ID：`ZXID`（事务ID）。由于ZAB协议要保证每一个消息严格的因果关系，因此必须将每一个事务Proposal按照其`ZXID`先后顺序来进行排序与处理：

### 1.3 具体流程

* Leader为每一个Follower服务器分配一个单独的队列，然后将需要广播的事务Proposal依次放入队列中，并且根据FIFO策略进行消息发送。
* Follower接受到Proposal之后，都会首先将其以事务日志的形式写入到本地磁盘中去，并且在成功写入后反馈给Leader一个Ack
* 当接收到超过半数Follower反馈Ack之后，就会广播一个Commit消息给所有Follower来通知事务提交，同时Leader自身提交该事务
* Follower收到`commit`之后进行事务提交

## 二、崩溃恢复
### 2.1 概述
* Leader崩溃
* 网络原因导致Leader失去与过半Follower的联系

则进入崩溃恢复模式。恢复过程结束后，还需要选举出新的Leader。Leader选举算法满足：
* 确保能够快速地选举出新的Leader
* Leader自身知道已经被选举为Leader
* 让集群中其他的机器快速感知到新的Leader服务器

### 2.2 基本特性

ZAB协议规定：
> 如果一个事务Proposal在一台机器上被处理成功，那么应该在所有的机器上都被处理成功，哪怕机器出现故障崩溃。

在崩溃恢复过程中可能会出现两种数据不一致的隐患：
#### 2.2.1 ZAB协议需要确保那些已经在Leader服务器上提交的事务最终被所有服务器都提交
![崩溃恢复1](https://raw.githubusercontent.com/zzzvvvxxxd/Picture/master/hexo/ZAB%E5%B4%A9%E6%BA%83%E6%81%A2%E5%A4%8D1.JPG)  
Leader(Server1)已经得到过半的`Ack`反馈，但是在它将`Commit`消息发送给所有Follower之前，Leader挂了。  
如上图，ZAB需要确保事务P2最终能够在所有的服务器上都被提交成功，否则将出现不一致。  
#### 2.2.2 ZAB协议需要确保丢弃那些只在Leader服务器上被提出的事务
![崩溃恢复2](https://raw.githubusercontent.com/zzzvvvxxxd/Picture/master/hexo/ZAB%E5%B4%A9%E6%BA%83%E6%81%A2%E5%A4%8D2.JPG)  
如果在崩溃恢复过程中出现一个需要被丢弃的提案，那么在崩溃恢复后需要跳过该事物。   
如上图，Leader(Server1)在提出P3后崩溃退出，其他机器都没有收到P3，于是当Server1恢复后，ZAB需要确保其丢弃事务P3。   

### 2.3 选举算法
以上决定了ZAB需要设计这一个Leader算法：
> 能够确保提交已经被Leader提交的事务Proposal，丢弃已经被跳过的Proposal

选举出来的Leader具有最高`ZXID`，保证了新的Leader一定具有所有已经提交的提案，也省去了检查Proposal的提交和丢弃的工作。  


## 三、数据同步
### 3.1 正常的数据恢复流程
Leader服务器会为每个Follower服务器准备一个队列，并将那些没有被各Follower服务器同步的事务以Proposal消息的形式逐个发送，并紧跟一个commit消息。等到Follower服务器将所有的尚未同步事务Proposal都从Leader服务器上同步过来并成功应用到内存数据库后，Leader服务器就会将该Follower加入到可用Follower列表中，开始后续流程。  

### 3.2 如何处理需要被丢弃的事务Proposal

`ZXID`:  
64位数字
* 低32位看成是一个简单的递增计数器，针对每一个客户端请求，Leader每生成一个Proposal就会进行加1
* 高32位代表Leader周期epoch的编号，每当选举产生一个新的Leader，epoch+1，低32位同时恢复为0

基于这样的策略，当一个包含了上一个Leader周期中没有提交的事务Proposal的服务器启动时，其肯定无法成为Leader，则以Follower角色连上当前的Leader服务器。Leader会根据自己服务器上最后提交的Proposal和Follower的Proposal进行比较，Leader会要求Follower进行一定的回退操作，回退到最后一个确认已经被过半机器提交的事务。
