##MySQL复制--基于binlog+position环境搭建

***首先启动数据库ls /data/mysql/mysql3306/start.sh 使用脚本启动数据库***

***首先主库库中的my.cnf配置***

    # cat /data/mysql/mysql3306/my3306.cnf |grep "server"
    server-id = 1003306
    
    # cat /data/mysql/mysql3306/my3306.cnf |grep log-bin
    log-bin = /data/mysql/mysql3306/logs/mysql-bin

    # cat my3306.cnf |grep updates
    log_slave_updates

2.在主库创建复制专用账号
    
    >create user 'repl'@'192.168.75.%' identified by 'repl4slave';
    
    >select user,host from mysql.user; ----查询用户是否创建成功
    
    >grant replication slave on *.* to 'repl'@'192.168.75.%';
    
    >root@localhost:mysql3306.sock [(none)]>show master status;
 

***从库配置***
    # cat my3307.cnf |grep log-bin
    log-bin = /data/mysql/mysql3307/logs/mysql-bin
    
    # cat my3307.cnf |grep server-id
    server-id = 1003307
    
    root@localhost:mysql3307.sock [(none)]>help change master to;
    CHANGE MASTER TO
      MASTER_HOST='master2.mycompany.com',
      MASTER_USER='replication',
      MASTER_PASSWORD='bigs3cret',
      MASTER_PORT=3306,
      MASTER_LOG_FILE='master2-bin.001',
      MASTER_LOG_POS=4,
      MASTER_CONNECT_RETRY=10;
  
    > change master to master_host='192.168.75.129', master_port=3306, master_user='repl', master_password='repl4slave',master_log_file='mysql-bin.000021',master_log_pos=441;

----------

    root@localhost:mysql3307.sock [(none)]>show slave status\G;<----##查看复制状态
    *************************** 1. row ***************************
       Slave_IO_State: 
      Master_Host: 192.168.75.129
      Master_User: repl
      Master_Port: 3306
    Connect_Retry: 60
      Master_Log_File: mysql-bin.000021
      Read_Master_Log_Pos: 441
       Relay_Log_File: relay-bin.000001
    Relay_Log_Pos: 4
    Relay_Master_Log_File: mysql-bin.000021
     Slave_IO_Running: No
    Slave_SQL_Running: No
    
    root@localhost:mysql3307.sock [(none)]>start slave;  <----##开始slave状态
    Query OK, 0 rows affected (0.00 sec)
    
    
    root@localhost:mysql3307.sock [(none)]>stop slave;  <----##停止slave状态
    Query OK, 0 rows affected (0.00 sec)


***针对复制线程开启关闭：`start slave io_thread; start slave sql_thread;`
					  `stop slave io_thread; stop slave sql_thread;`
***
    root@localhost:mysql3307.sock [(none)]>show slave status\G;
    *************************** 1. row ***************************
      Slave_IO_State: Waiting for master to send event
      Master_Host: 192.168.75.129
      Master_User: repl
      Master_Port: 3306
      Connect_Retry: 60
      Master_Log_File: mysql-bin.000021
      Read_Master_Log_Pos: 441
      Relay_Log_File: relay-bin.000002
      Relay_Log_Pos: 320
      Relay_Master_Log_File: mysql-bin.000021
      Slave_IO_Running: Yes
      Slave_SQL_Running: Yes
			


  4.登录主库
# `mysql -S /tmp/mysql3306.sock`
    >show variables like "%server_id%";
    >show variables like "%log%";

5.校验数据的一致性。

    >create database db2;
    Query OK, 1 row affected (0.00 sec)

    >show databases;
    
    >use db2;

    >create table t1(id int);   <----##创建t1表。
    
    >insert into t1 values(1); <-----##在t1表中插入数据“1”
    root@localhost:mysql3306.sock [db2]>select * from t1;

在从库中查看

`> select * from t1;  <----看到和主库的数据是一样的`。


----一下是扩展问题----
***扩展问题
###在搭建slave库前主库中已经有了一些数据了。解决方法###

    mysqldump  导出数据  --single-transaction --master-data=2
    cd /data
    mysqldump -uroot -p --single-transaction --master-data=2 -A >zst-2017.5.23.sql   ##<----备份成功到data目录##

`scp zst-2017.5.23.sql 192.168.75.129:/data/mysql/mysql3307/data/	`##<----scp拷贝到远程计算机目录##

----以上是主库操作----

----以下是从库操作----

    >stop slave;			<----停止复制
    >reset slave all;		<----清空slave数据
# tail -f zst-2017.5.23.sql
	……………………
	-- Dump completed on 2017-05-23 13:13:37  ----看到这句提示就是完整的数据

    # mysql -uroot -p <zst-2017.5.23.sql   恢复之前远程拷贝的数据文件

***假如在同一台机器多个实例恢复时候使用加sock文件

    # mysql -S /tmp/mysql3307.sock -p <zst-2017.5.23.sql
***

回复完之后more一下备份文件
`# more zst-2017.5.23.sql` 

-- CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.000021', MASTER_LOG_POS=2387; ----找到这条语句

    > change master to master_host='192.168.75.129', master_port=3306, master_user='repl', master_password='repl4slave',master_log_file='mysql-bin.000021',master_log_pos=2387;
    
    > reset master;
    Query OK, 0 rows affected (0.00 sec)
    > show slave status;
    NO
    NO
    > start slave;
    > show slave status;
    YES
    YES
##在主库中插入数据测试一下
    use zst；insert into t1 values(1); select * from t1; 
##在从库中查看
    use zst；select * from t1;
##同步成功就ok啦。就完成了基于binlog+position环境搭建
***











