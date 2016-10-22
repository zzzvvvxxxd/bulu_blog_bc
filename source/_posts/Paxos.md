title: 【一致性协议03】 Paxos和分布式系统
date: 2016-08-08 21:43:21
categories: 分布式
tags: [分布式,一致性协议]
---

## 资源汇总
* [Paxos Made Simple](http://research.microsoft.com/en-us/um/people/lamport/pubs/paxos-simple.pdf)
* [使用Basic-Paxos协议的日志同步与恢复](http://mp.weixin.qq.com/s?__biz=MzA4MzYxMjEwMg==&mid=203432378&idx=1&sn=34d280eb89d6bad352fdbbf856d86c99#rd)
* [可靠分布式系统基础 Paxos 的直观解释](http://drmingdrmer.github.io/tech/distributed/2015/11/11/paxos-slide.html)
* [paxos和分布式系统(视频)](http://www.tudou.com/programs/view/e8zM8dAL6hM/)

## Paxos和分布式存储系统（问题定义）
Paxos用来确定一个不可变变量的取值
* 取值可以是任意二进制数据
* 一旦确定将不再更改，并且可以被获取到（不可变性、可读取性）

在分布式存储系统中应用Paxos
* 在分布式存储系统中，数据本身可变，采用多部分进行存储
* 多个副本的更新操作序列`[op1, op2, op3, ... , opn]`是相同的、不变的。
* 用Paxos依次来确定不可变变量`opi`的取值（即第i个操作时什么）
* 每确定完Opi之后，让各个数据副本执行`opi`

将问题转换为，设计一个系统，来存储名称为var的变量
> * 系统内部由多个`Acceptor`组成，负责存储和管理`var`变量
* 外部有多个`proposer`机器任意并发调用API，向系统提交不同的`var`取值
* var可以是任意二进制数据
* 系统对外的API库接口为：
  * `propose(var, V)`返回`<ok, f>`或者`<error>`（f是系统内部已经确定的取值，如果propose设置成功则f为V，否则为其他`proposer`设置成功的结果）

系统要保证var的取值满足一致性
* 如果`var`的取值没有确定，则var的取值为`null`
* 一旦`var`的取值被确定，则不可更改。并且一直可以获取到这个值

系统要满足一定的容错特性
* 可以容忍任意`proposer`机器出现故障
* 可以容忍少数`Acceptor`故障（半数以下）

暂时不考虑：
* 网络分化
* `Acceptor`故障会丢失`var`

## 难点
分布式环境下确定一个不可变变量的难点有：
* 管理多个`Proposer`的并发执行
* 保证`var`变量的不可变性
* 容忍任意`Proposer`机器故障
* 容忍半数以下`Acceptor`机器故障

## 方案1
先考虑由单个`Acceptor`组成，通过类似**互斥锁机制**来管理并发的`Proposer`运行
* `Proposer`首相向`Acceptor`申请`Acceptor`的互斥访问权限，然后才能请求`Acceptor`接受自己的取值
* `Acceptor`给`Proposer`发放互斥访问权，谁申请到互斥访问权，就接受谁提交的值

这样就让`Proposer`按照获取互斥访问权的顺序依次访问`Acceptor`，一旦`Acceptor`接收了某个`Proposer`的取值，则认为`var`取值被确定
，其他`Proposer`不再更改。

基于互斥访问权的`Acceptor`的实现
```
Acceptor保存变量var和一个互斥锁lock

Acceptor::prepare():
    加互斥锁，给予var的互斥访问权，并返回当前的var取值f

Acceptor::relaese():
    解互斥锁，收回var的互斥访问权

Acceptor::acceptor(var, V)
    如果已经加锁，并且var没有取值，则设置var为V，并且释放锁
```
Proposer(var, V)的两阶段实现
```
第一阶段：通过Acceptor::prepare()获取互斥访问权和当前var的取值
    如果不能，返回<error>（锁被其他Proposer占用）

第二阶段：根据当前var的取值f，选择执行
    如果f为null，则通过Acceptor::accept(var, V)提交数据V
    如果f不会null，则通过Acceptor::release()释放访问权，返回<ok,f>
```
**缺陷**：
> 如果Proposer在释放互斥锁之前发生故障，会导致系统陷入死锁。（不能容忍任何Proposer机器故障）


## 方案2
引入抢占式访问权
`Acceptor`可以让某个`Proposer`获取到的访问权失效，不再接收它的访问。之后，可以将访问权交给其他`Proposer`，让其他`Proposer`访问`Acceptor`
`Proposer`向`Acceptor`申请访问权时，指定编号`epoch`，并认为越大的`epoch`值越新，获取到访问权后，才能向`Acceptor`提交值。
`Acceptor`采取**喜新厌旧**原则：
* 一旦接收到更新的`epoch`申请，马上让之前的旧`epoch`访问权限失效，不再接收它们提交的取值
* 给新的`epoch`发放访问权，只接受新`epoch`提交的取值

也就是新的`epoch`可以抢占旧的`epoch`，为了保持一致性，不同`epoch`的`Proposer`之间采***后者认同前者***的原则。
1. 肯定旧的`epoch`无法生成确定性取值时，新的`epoch`会提交自己的value，不会冲突
2. 一旦旧`epoch`形成了确定性取值，新的`epoch`可以获取到该值，并会认同，而不会破坏

基于抢占式访问权的`Acceptor的实现`
```
Acceptor保存的状态
    当前var的取值<accepted_epoch, accepted_value>
    最新发放访问权的epoch(lastest_prepared_epoch)

Acceptor::prepare(epoch)
    只接受比latest_prepared_epoch更大的epoch，并给予访问权
    记录latest_prepared_epoch = epoch；返回当前var取值

Acceptor::accept(vae, prepared_epoch, V)
    验证latest_prepared_epoch == prepared_epoch,如果不等则说明已经有一个更新的epoch抢占了访问权
    设置var的取值<accepted_epoch, accepted_value> = <prepared_epoch, V>
```
Propose(var, V)的两阶段实现
```
第一阶段：获取epoch轮次的访问权和当前var的取值
    简单选取当前时间戳为epoch，通过Acceptor::prepare(epoch)，获取epoch轮次的访问权和当前的var
    如果不能获取返回<error>，Proposer运行失效

第二阶段：采用"后者认同前者"原则执行
    1. 如果var的取值为null，则肯定旧epoch无法生成确定性取值，则通过Acceptor::accept(var, epoch, V)提交数据。
       成功后返回<ok, V>
       如果accept失败，则返回<error>(被新的epoch抢占，或者Acceptor故障)
    2. 如果var取值存在，则此确定值肯定是确定性取值，此时认同它不再更改，直接返回<ok, accepted_value>
```
**总结**：
基于抢占式访问权的核心思想让Proposer将epoch递增的顺序抢占式的依次运行，后者会认同前者。
【优点】：
可以避免方案一种Proposer故障带来的死锁问题，并且仍可以保证var取值的一致性
【缺点】：
单点问题任然存在

## Paxos
Paxos就是在方案二的基础上，试图解决`Acceptor`的单点问题，引入了多`Acceptor`。仍然采用喜新厌旧的原则，但是
因为引入了多个`Acceptor`，则不再简单地"后者认同前者"，还采用了"少数服从多数"原则：
> 一旦某个epoch的取值f被半数以上的Acceptor接受，则认为此var的取值确定为f，不再更改。

Paxos中的`Acceptor`实现相较方案二保持不变。

```
Propose(var, V)第一阶段： 选定epoch，获取epoch访问权和对应的var取值
    获取半数以上Acceptor的访问权和对应一组var取值
    和方案二一致

Propse(var, V)第二阶段： 采用后者服从前者的原则
    # 在肯定旧epoch无法生成确定性取值时，新的epoch会提交自己的取值，不会冲突
    # 一旦旧epoch形成确定性取值，新的epoch肯定可以获取到此取值，并且会认同此取值，不会破坏
    1. 如果获取的var都为空，则旧epoch无法形成确定性取值，此时努力使<epoch, V>成为确定性取值
        * 向epoch对应的所有Acceptor提交取值<epoch, V>
        * 如果收到半数以上的成功，则返回<ok, V>
        * 否则返回<error>（新epoch抢占或者Acceptor故障）
    2. 如果获取的var取值存在，直接认同最大accepted_epoch对应的取值f，努力使<epoch, f>成为确定性取值
        * 如果f出现半数以上，则说明f已经是确定性取值，直接返回<ok, f>
        * 否则，向epoch对应的所有Acceptor提交取值<epoch, f>
```

**总结**：
核心思想：在抢占式访问（方案2）的基础上引入了多Acceptor，保证一个epoch，只有一个Proposer运行，Proposer按照递增的
次序运行。
新epoch的Proposer采用后者认同前者的思路运行。

> Paxos算法可以满足容错性要求。
* 半数以下的Acceptor出现故障时，存货的Acceptor依然可以生成var的确定性取值
* 一旦var的值被确定，即使出现半数以下的Acceptor故障，此取值可以被获取，且不再被更改。

## Paxos算法的Liveness问题
新轮次的抢占会让旧轮次停止运行，如果每一轮在第二阶段执行成功之前都被新一轮次抢占，则导致活锁。如何解决？
