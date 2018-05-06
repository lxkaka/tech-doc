## AWS RDS主从同步延迟调查
### 背景
我们的关系型数据库用的是AWS的RDS，mysql主从延迟这个问题之前是知道的，简单怀疑是AWS的主从同步机制问题但是一直没有重视起来去研究一下问题的根源。终于，隐患暴露处来--- 一次`alter table`和主从延迟曾大发生的时间点重合，从而加剧主从延迟。造成从库远远落后于主库，而数据不一致。必须找到延迟过大的根本原因，解决这个问题。
在分析问题之前我们先简要介绍一下mysql主从同步机制
### [主从同步机制](https://dev.mysql.com/doc/refman/5.6/en/replication-implementation-details.html)
mysql在主库上记录binlog。在准备提交事务完成数据更新前，主库把数据更新的事件记录到binlog中。
mysql通过上面说的binlog用3个线程来实现主从同步，master上的`Binlog Dump Thread`, slave上的`Slave I/O Thread`和`Slave SQL Thread`

  * Slave I/O Thread: I/O线程跟主库建立一个普通的客户端连接，并请求从指定日志文件的指定位置（或者从最开始的日志）之后的日志内容, 存到relaylog中, 然后将读取到的主库binlog的文件名和位置记录到master-info文件中
  * Binlog Dump Thread: 读取主库上的binlog中的事件，根据请求信息读取制定日志指定位置之后的日志信息，返回给`Slave I/O Thread`。返回信息中除了日志所包含的信息之外，还包括本次返回的信息已经到Master端的binlog文件的名称以及binlog的位置。
  * Slave SQL Thread: 从relaylog中读取事件并且在从库执行，从而实现从库的更新
  
  ![replica](http://7xorjs.com1.z0.glb.clouddn.com/mysql-replica.jpg)
    

通过观察从库的同步延迟曲线发现每天固定的三个时间点从库都会报警主从延迟超过阈值。
   
![replica_lag](http://7xorjs.com1.z0.glb.clouddn.com/replica-lag.jpg)
### 首先从AWS官方文档上寻找主从同步延迟的[办法](https://docs.aws.amazon.com/zh_cn/AmazonRDS/latest/UserGuide/USER_ReadRepl.html)
> 可通过多种方式来减少对源数据库实例的更新与对只读副本的后续更新之间的滞后，例如：
将只读副本的存储大小和数据库实例类调整到与源数据库实例类似。
确保源数据库实例和只读副本使用的数据库参数组中的参数设置相兼容。有关更多信息和示例，请参阅本部分后面的有关 max_allowed_packet 参数的讨论。
Amazon RDS 监控只读副本的复制状态，如果由于任何原因停止复制，则将只读副本实例的 Replication State 字段更新为 Error。可能会有这样的例子，在您的只读副本上运行的 DML 查询与对源数据库实例的更新冲突。
您可通过查看 Replication Error 字段，检查 MySQL 或 MariaDB 引擎引发的关联错误的详细信息。还生成指示只读副本状态的事件，包括 RDS-EVENT-0045、RDS-EVENT-0046 和 RDS-EVENT-0047。有关这些事件和事件订阅的详细信息，请参阅 使用 Amazon RDS 事件通知。如果返回 MySQL 错误消息，则检查 MySQL 错误消息文档中的错误编号。如果返回 MariaDB 错误消息，则检查 MariaDB 错误消息文档中的错误。
一个可导致复制出错的常见问题是只读副本的 max_allowed_packet 参数的值小于源数据库实例的 max_allowed_packet 参数的值。max_allowed_packet 参数是可在数据库参数组中进行设置的自定义参数，用于指定可在数据库上执行的最大 DML 代码大小。有时候，与源数据库实例关联的数据库参数组中的 max_allowed_packet 参数值，要小于与源的只读副本关联的数据库参数组中的 max_allowed_packet 参数值。在这些情况下，复制过程可能会引发错误 (数据包大于 'max_allowed_packet' 字节) 并停止复制。可通过让源和只读副本使用 max_allowed_packet 参数值相同的数据库参数组来纠正该错误。

按照官网文档的建议尝试，发现并不是以上所说问题。必须深入分析，找到原因。

### 分析
1. 是否在延迟报警的时间点有大量的写操作
  通过代码排查，在这些时间点并没有类似的操作
2. 在延迟发生的时间点观察执行状态  
    主库`show PROCESSLIST`，并没有很多的写操作同时也印证了代码的排查的结论  
    从库`show PROCESSLIST`，看到十分重要的信息 发生了**锁表**并且有很多写入在排队    
    
    ![table_lock](http://7xorjs.com1.z0.glb.clouddn.com/table_lock.png)
3. 观察`innodb_trx`和`innodb_lock`
  查看是否有大的事务拿着锁不放，结果是没有
通过以上信息，判读不是我们自己的写操作造成主从同步延迟  
继续寻找线索
4. 查看发生延迟时间点的慢查询日志
  查看到这个时候有很多类似的查询  
  `SELECT /*!40001 SQL_NO_CACHE */ * FROM`
  先来看这个查询的意义：
  * /*!   */ 这是mysql里的语法，表示达到条件会执行相应的语句。  
  * !后面是版本号， 如果本数据库等于或大于此版本号，那么语句会执行。  
  * 那么这句话的意思是 如果版本号大于或等于4，会执行 sql_no_cache, 就是不用缓存数据。 而并非说本次查询不作为下次查询的缓存。  
  * 在备份操作时Mysql 会自动调用此语法。该语句会查询到表中所有数据，在备份文件中会生成相应的insert语句。
5. 查看数据库备份
  每隔8小时执行一次
  ```
  mysqldump -u ${user} -p${password} -h ${host} ${name} > ${filename}
  ```
  
  ![db_backup](http://7xorjs.com1.z0.glb.clouddn.com/db_backup.png)

  可以看到时间点与主从同步延迟的报警时间完全吻合  

### 原因
通过一系列列大胆假设和小心求证,可以断定是我们在执行**自动备份的时候大量锁表**，造成很多同步的写入操作被阻塞  

* `mysqldumb` 有个很关键的点是为了保证同步后数据的一致性，会采用 `lock-tables`, `lock-all-tables`, `single-transaction`不同的机制。 
   
  `lock-tables`: 默认会选择这个机制，把所有需要dump的表都会加锁，当然写入表的操作就被阻塞了  
    
  `lock-all-tables`: dump期间锁定所有数据库中的所有表，以保证数据的一致性。这是一个全局锁定，并且自动关闭–single-transaction 和–lock-tables 选项。这个参数副作用比较大，这是全库锁定，备份执行过程中，该库无法进行读写操作 
   
  `single-transaction`: 这个机制把事务隔离级别设置为可重复读，在dump之前提交一个`START TRANSACTION`语句，不会阻塞任何应用程序且能保证导出时数据库的一致性状态。它只适用于多版本存储引擎(MVCC)，仅InnoDB。本选项和–lock-tables 选项是互斥的，因为LOCK TABLES 会使任何挂起的事务隐含提交，使用参数–single-transaction会自动关闭该选项。   

### 解决方案
* 因为我们的db引擎是innodb, 最简单的方式就是修改`mysqldump`保证数据一致性的机制，采用`single-transaction` 
	 
	`mysqldump -u ${user} -p${password} -h ${host} ${name} --single-transaction --quick > ${filename}`

	`single-transaction`只适用于innodb, 在innodb导出时会建立一致性快照，在保证导出数据的一致性前提下，又不会堵塞其他会话的读写操作，相比–lock-tables参数来说锁定粒度要低，造成的影响也要小很多。指定这个参数后，其他连接不能执行ALTER TABLE、DROP TABLE 、RENAME TABLE、TRUNCATE TABLE这类语句，事务的隔离级别无法控制DDL(Data Definition Languages)语句。**所以改表的时候，要注意查看是否有数据库备份执行**

* 再起一个从库，单独在此从库上执行备份
  因为数据库备份的时候占用不少资源，而同时我们的从库也有大量的读的操作，会互相影响。
* 不用mysqldump选择别的备份方案, 比如`XtraBackup`

下图是执行解决方案后从库延迟的曲线图，可以看到之前定时出现的主从延迟(心头大患)已经消失了(开心.jpg)

![new_lag](http://7xorjs.com1.z0.glb.clouddn.com/replica_no_lag.png)

主要参考：https://dev.mysql.com/doc/refman/5.6/en/replication.html。
### 总结
多学习多思考，对数据库的操作包括基本的增删改查要有敬畏之心。

附加部分
## 数据库查询优化
数据库主从同步延迟报警的时间点，还有主库CPU和连接数报警，在一开始因为时间点基本一致，而认为两者一定时相关的，误导了我对根本原因的判读。更重要的是针对这种报警必须优化。下面就是一个优化的例子。
### 1. 分析原因
根据数据库报警信息 CPU和connections超过阈值报警，找到对应时间点的慢查询  

  * 根据报警时间点，找到对应的定时任务
  * 根据时间，查看慢查询日志
    AWS RDS开启slow_query_log，设置三个关键参数:   
    
	 * `slow_query_log`（1）, 
    * `long_query_time`（比如10秒）, 
    * `log_output`（file)  
       
    设置完成后可以用`show variables like '参数名称'`来查看是否设置成功
  
  ```
  # Time: 180504 0:28:20
  # User@Host: xxx(脱敏) Id: 164478982
  # Query_time: 3.542192 Lock_time: 0.000056 Rows_sent: 1 Rows_examined: 76281
  SET timestamp=1525393700;
  SELECT COUNT(*) AS `__count` FROM `core_membership` WHERE (`core_membership`.`business_group_id` = 2908 AND `core_membership`.`last_visited_at` > '2018-05-04 23:59:59.999999' AND `core_membership`.`last_visited_at` <= '2018-02-03 23:59:59.999999');

  # 这个慢查询必须优化
  # Time: 180504 0:27:31
  # User@Host: xxx(脱敏) Id: 164477983
  # Query_time: 45.261294 Lock_time: 0.000043 Rows_sent: 1 Rows_examined: 8583
  SET timestamp=1525393651;
  SELECT COUNT(*) AS `__count` FROM `core_membership` WHERE (`core_membership`.`business_group_id` = 1976 AND `core_membership`.`level` = 1);
  ```
  慢查询和应用中的定时任务要执行的查询完全一致
### 2. 优化
分析慢查询日志确定拖垮db的根本原因是上面的第二条查询
通过`explain`分析上面的查询语句，虽然用到了索引，但依然很慢

![explain](http://7xorjs.com1.z0.glb.clouddn.com/explain_sql.png)
  **原因**：level为1的会员占了绝大多数，即使用了索引，这样的查询扫索引依然很多行，所以慢
  下面的执行结果非常明显
  
  ```
  sql> SELECT COUNT(*) AS `__count` FROM `core_membership` WHERE (`core_membership`.`business_group_id` = 2708 AND `core_membership`.`level` = 1)
[2018-05-04 13:16:31] 1 row retrieved starting from 1 in 2s 987ms (execution: 2s 964ms, fetching: 23ms)
sql> SELECT COUNT(*) AS `__count` FROM `core_membership` WHERE (`core_membership`.`business_group_id` = 2708 AND `core_membership`.`level` > 1)
[2018-05-04 13:17:16] 1 row retrieved starting from 1 in 598ms (execution: 587ms, fetching: 11ms)
  ```
  **优化方案**：level大于1的会员为少数，最简单有效的方法就是只查询高等级的会员，这样查询索引的时候自然就快。然后再用总数跟高等级的人数做减法，来避免level 1的查询
  下图是优化后的主库CPU占用曲线图，可以看到优化后峰值明显减少
  
  ![opt-query](http://7xorjs.com1.z0.glb.clouddn.com/opt-query.png)
  图上剩下的峰值就是下一步的优化对象。  

