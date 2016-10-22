title: 【mysql】binlog详解(1)
date: 2016-02-13 16:11:22
categories: 数据库
tags: [数据库,mysql]
---
>The binary log contains “events” that describe database changes such as table creation operations or changes to table data. It also contains events for statements that potentially could have made changes (for example, a DELETE which matched no rows), unless row-based logging is used. The binary log also contains information about how long each statement took that updated data.

binlog是mysql的二进制日志，记录了mysql数据的更新或者潜在的更新记录，同时还包含了每条语句的执行耗时和更新时间。  
注意： binlog主要有`statement-based`、`row-based logging`和`mixed logging` 三种格式，row-based的记录中不包括潜在更新记录。

binlog有两个主要的功能：
* 用于复制
> master发送了包含了所有events的binary log给slaves，slaves执行binary log中的event来保证和master之间的数据一致性

* 一些特定的数据恢复操作，会使用binlog

## 1. binlog相关的启动参数

```
--binlog-row-event-max-size=N
```
指定行格式复制日志event的最大大小，单位为byte，每一行数据会被切分到多个小于该限制的event包中，必须是256byte的倍数。  
* 默认值：8192
* min：256
* max：18446744073709551615（64位平台）

```
--log-bin[=base_name]
```
开启binary log功能（用于复制和备份），指定日志名称的base name，mysql会使用指定的base name作为前缀，连续的数据作为后缀生成一系列的日志文件。默认会使用`host_name`作为base name。  

```
--log-bin-trust-function-creators[={0|1}]
```
指定mysql如何处理函数及存储过程的创建，取决于用户是否认为自己的存储过程及函数是否安全（确定的或者不修改数据），默认对于不安全的存储过程及函数不会写入binlog。这也就是，在开启binlog的时候，如果不设置该参数，使用自定义函数时会出现错误的原因。

## 2. Statement selection options
下面介绍的选项会影响写入到binary log中的记录语句，因此会控制master发送到slaves中的日志的内容。同样，slaves也有相应的参数，在日志选择哪些命令可以执行。
### 2.1  `--binlog-do-db`
```
--binlog-do-db=db_name
```
这个参数的影响取决是使用`statement-based`还是`row-based logging`格式的日志。  
**statement-based logging**  
只有操作默认数据库（USE语句指定）的语句才会被记录。如果要指定多个数据库，需要重复使用该参数。但是跨数据库的操作
不会因此被记录，因为这里的检查原则，mysql为了尽可能的简单，只会去检查最近的`USE`语句指定的数据库是否在该参数指定的数据库中。例如：
```
USE sales;
UPDATE prices.discounts SET percentage = percentage + 10;
```
该语句在参数为`--binlog-do-db=sales`的情况下**会**被记录进binlog的，虽然看起来不太能理解。  
再举个例子：
```
USE prices;
UPDATE sales.january SET amount=amount+1000;
```
该语句在参数为`--binlog-do-db=sales`的情况下是**不会**被记录进binlog的。  
**Row-based logging**  
日志会被严格限定在`db_name`所指定的数据库上，不再受`USE`的影响。  

在跨数据的修改操作上，需要举一个具体的例子来加深说明`statement-based`和`row-based logging`两者的区别，假设初始参数为`binlog-do-db=db1`：
```
USE db1;
UPDATE db1.table1 SET col1 = 10, db2.table2 SET col2 = 20;
```
这条操作`statement-based`的情况下，对两个表的`UPDATE`操作都被记录，如果是`row-based logging`，只有针对table1的操作会被记录。
现在假设更换默认数据库为db4  
```
USE db4;
UPDATE db1.table1 SET col1 = 10, db2.table2 SET col2 = 20;
```
`statement-based`不会记录这条操作，而`row-based logging`则会记录table1的`UPDATE`操作日志。


### 2.2  `--binlog-ignore`
```Java
--binlog-ignore-db=db_name
```


这个参数看起来很好理解，但是要注意的是，`CREATE TABLE`和`ALTER TABLE`不会受其影响都一定会被写入log，一般这个参数影响的是`UPDATE`操作。  




### 2.3 ` --binlog-format`
指定binlog使用的格式，可选：statement、row、mixed，具体参考下一篇日志




> 更多的配置参数可以参考[1]中的官方说明。  

## 相关系统参数和配置
### `sync_binlog`
binlog刷新到磁盘的时机跟sync_binlog参数相关，如果设置为0，则表示MySQL不控制binlog的刷新，由文件系统去控制它缓存的刷新，而如果设置为不为0的值则表示每sync_binlog次事务，MySQL调用文件系统的刷新操作刷新binlog到磁盘中。设为1是最安全的，在系统故障时最多丢失一个事务的更新，但是会对性能有所影响，一般情况下会设置为100或者0，牺牲一定的一致性来获取更好的性能。

### `expire_logs_days`
指定binlog保留时间

### 清理binlog
要手动清理binlog可以通过指定binlog名字或者指定保留的日期  



```  
purge master logs to BINLOGNAME;
purge master logs before DATE;
```



### 查看binlog情况

```
SHOW MASTER LOGS;

+------------------+-----------+
| mysql-bin.000018 |       515 |
| mysql-bin.000019 |       504 |
| mysql-bin.000020 |       107 |
+------------------+-----------+
```



第一列是binlog文件名，第二列是binlog文件大小

## binlog和redo/undo log的区别
两者是完全不同的日志，主要有一下2个区别：
* 层次不同。redo/undo log是innodb层维护的，而binlog是mysql server层维护的，跟采用何种引擎没有关系，记录的是所有引擎的更新操作的日志记录。
* 记录内容不同。redo/undo日志记录的是每个页的修改情况，属于物理日志+逻辑日志结合的方式，目的是保证数据的一致性。binlog记录的都是事务操作内容，格式是二进制的。


## 参考：
1. [mysql文档 18.1.6.4 Binary Logging Options and Variables](http://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html)
2. [mysql文档 6.4.4 The Binary Log](http://dev.mysql.com/doc/refman/5.7/en/binary-log.html)  
