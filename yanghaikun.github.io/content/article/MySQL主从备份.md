+++
date = "2016-01-14T19:48:22+08:00"
draft = true
title = "MySQL主从备份读写分离设置"

+++

## MySQL主从备份读写分离设置
### 1.主从备份的优势，MySQL文档是这么写的

	Advantages of replication in MySQL include:
	Scale-out solutions - spreading the load among multiple slaves to improve performance. 
	In this environment, all writes and updates must take place on the master server. Reads, however, 
	may take place on one or more slaves. This model can improve the performance of writes 
	(since the master is dedicated to updates), while dramatically increasing read speed across an increasing number of slaves.

	Data security - because data is replicated to the slave, and the slave can pause the replication process, 
	it is possible to run backup services on the slave without corrupting the corresponding master data.

	Analytics - live data can be created on the master, while the analysis of the information can take place 
	on the slave without affecting the performance of the master.

	Long-distance data distribution - if a branch office would like to work with a copy of your main data, 
	you can use replication to create a local copy of the data for their use without requiring permanent access to the master.

##### 翻译成以下4点：
	1.读写分离，通过扩展slave来提高性能，master来负责插入和更新，查询可以扩展到多台slaves
	2.数据更有保证，不再需要获取master上一致的数据而可以基于slave上的数据直接启动应急的服务器
	3.数据分析，基于salve的数据进行分析，不用影响线上性能
	4.分布式远程数据，如果有人要基于你们的数据进行工作，可以让他们接触slave数据而不用操作master


### 2.言归正传，怎么配置和使用呢
###### 1.基于二进制日志文件
###### 2.MySQL5.6.5之后可以基于GTID（全局事务标示符）配置
##### 基于binary logging mechanism，就是master的数据有变化时，会写入binary log，slaves会根据配置去读取binary log，然后在本地数据库执行log里面的操作
	Important
	You cannot configure the master to log only certain events.
	不能配置master只记录某种事件
##### 配置步骤
###### 1.配置master，在主服务器的mysql配置文件中添加以下内容
	[mysqld]
	server-id=1
	log-bin=mysqlmaster-bin.log
	#For the greatest possible durability and consistency in a replication setup using InnoDB with transactions, you should use innodb_flush_log_at_trx_commit=1 and sync_binlog=1 in the master my.cnf file.
	innodb_flush_log_at_trx_commit=1
	sync_binlog=1
###### 重启master上mysql

###### 2.配置slave
	[mysqld]
	server-id=2
	#下面两个参数不是必须的，只有当你需要使用slave的log文件或者需要作为其他slave的master
	log-bin=mysqlslave-bin
	innodb_flush_log_at_trx_commit=1
	sync_binlog=1
###### 重启slave

###### 3.在master上进行一系列操作
1.创建用于主从复制的账户

	GRANT REPLICATION SLAVE ON *.* TO'repl'@'192.168.100.3' IDENTIFIED BY 'repl';
	或者
	CREATE USER 'repl'@'%.mydomain.com' IDENTIFIED BY 'slavepass';
	GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%.mydomain.com';
2.登录master上mysql，flush写数据并锁定表，

	mysql>FLUSH TABLES WITH READ LOCK;
3.然后执行以下语句，获取当前的二进制log文件名和位置

	mysql>SHOW MASTER STATUS;
![](/images/show_master_status.png)
在这个例子中二进制文件是mysqlmaster-bin.000001，位置是120，记录这两个值，稍后会用到，如果这个命令没有查询出数据，那么稍后用到的值就是空字符串和4

4.在master上使用mysqldump命令创建一个数据快照

	mysqldump -uroot -p --all-databases --master-data > dbdump.db
	当然也可以使用以下命令远程操作
	mysqldump -uroot -p -h127.0.0.1 -P3306 --all-databases --master-data > dbdump.db
5.解锁刚才锁定的数据库

	mysql>UNLOCK TABLES;

###### 4.操作从库

(1) 导入刚才创建的主数据库的快照

	上传dbdump.db，执行命令
	mysql -uroot -p < dbdump.db
	也可以远程执行
	mysql -uroot -p -hhost -P3306 < dbdump.db
(2) 在数据库中配置要复制的主库的信息
	
	mysql> CHANGE MASTER TO  MASTER_HOST='master_host_name', 
    MASTER_USER='replication_user_name', 
    MASTER_PASSWORD='replication_password', 
    MASTER_LOG_FILE='recorded_log_file_name', 
    MASTER_LOG_POS=recorded_log_position;
(3)	启动slave的复制线程，并查询slave的状态
	
	mysql> START slave;
	mysql> SHOW slave STATUS;
	#如果以下两个状态都为yes，说明主从配置成功
	Slave_IO_Running:	 Slave_SQL_Running:
 	Yes 				 Yes
###### 然后就可以在master上修改数据进行测试了。

#### 如果最后不成功，那么查看mysql的日志文件，看错误在哪里。
#### PS：windows下mysql的配置文件在此目录下C:\ProgramData\MySQL\MySQL Server 5.6


	


	

