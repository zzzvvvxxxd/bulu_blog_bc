title: 【mysql】binlog详解(2) binlog格式
date: 2016-06-22 09:44:17
categories: 数据库
tags: [数据库,mysql]
---

可以指定三种binary log的格式（启动时指定）：  
```
--binlog-format=STATEMENT
--binlog-format=ROW
--binlog-format=MIXED
```  
* `statement-based logging`： 基于SQL语句，Mysql5.6默认，某些语句和函数如UUID, LOAD DATA INFILE等在复制过程可能导致数据不一致甚至出错。  
* `row-based logging`：基于行，记录影响table中每一行的事务，很安全，很安全。但是binlog会比其他两种模式大很多，在一些大表中清除大量数据时在binlog中会生成很多条语句，可能导致从库延迟变大。  
* `mixed logging`：使用statement-based logging作为默认，但是日志模式可能会在某些情况下自动切换到row-based logging。  

`statement-based logging`可能回带来复制上的安全问题：
> With statement-based replication, there may be issues with replicating nondeterministic statements. In deciding whether or not a given statement is safe for statement-based replication, MySQL determines whether it can guarantee that the statement can be replicated using statement-based logging. If MySQL cannot make this guarantee, it marks the statement as potentially unreliable and issues the warning, Statement may not be safe to log in statement format.  
You can avoid these issues by using MySQL's row-based replication instead.

除了开头提到的启动参数，binlog的格式还可以在运行时切换：  
```
# GLOBAL
mysql> SET GLOBAL binlog_format = 'STATEMENT';
mysql> SET GLOBAL binlog_format = 'ROW';
mysql> SET GLOBAL binlog_format = 'MIXED';

# SESSION
mysql> SET SESSION binlog_format = 'STATEMENT';
mysql> SET SESSION binlog_format = 'ROW';
mysql> SET SESSION binlog_format = 'MIXED';
```

一般基于以下几个理由会设置`SESSION`级别的binlog format切换：
* 在对数据库做出了很多小的改变时，可能会需要使用 row-based logging  
* 在使用`WHERE`进行更新操作时，可能会影响很多行记录，使用statement-based logging来记录少量的事务语句日志，会比记录很多行的改动有效得多。  
* 有些语句可能需要执行很长时间，但是实际只改动几行记录。使用row-based logging会对复制功能比较友好。  


有些情况切换logging format可能会返回`error`:
1. 在使用InnoDB时，如果隔离级别是`READ COMMITTED`或`READ UNCOMMITTED`，只有row-based format可以使用。切换到statement-based其实也是可行的，但是很加就会导致错误，因为这种情况下InnoDB就无法插入数据  
2. 临时表存在时，不推荐切换logging format
>Switching the replication format at runtime is not recommended when any temporary tables exist, because temporary tables are logged only when using statement-based replication, whereas with row-based replication they are not logged. With mixed replication, temporary tables are usually logged; exceptions happen with `user-defined functions (UDFs)` and with the `UUID()` function.


注意：  
`ROW`格式下依然会有部分的语句按照`STATEMENT`格式记录，例如所有的DDL语句：`CREATE TABLE`, `ALTER TABLE`和 `DROP TABLE`
