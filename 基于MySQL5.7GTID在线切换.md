## 几种状态的说明：

1. OFF			   不产生GTID，基于Binlog+position，slave也不能接受GTID的日志
2. OFF_PERMISSIVE  不产生GTID,但做为Slave可以识别GTID事务也可以识别非GTID事务
3. ON_PERMISSIVE   产生GTID事务，Slave可以处理GTID事务和非GTID事务
4. ON		       产生GTID事务，Slave只接受GTID的事务

----------
目前的环境：

192.168.11.100 3306 GTID

192.168.11.101 3306 GTID

实验一：把GTID的环境切换成非GTID

切换流程：
首先禁掉GTID。。。

    1. show slave status\G;
    2. stop slave;

    3.change master to master_host='192.168.11.100',master_port=3306,master_user='repl',
	master_password='repl4slave',
	master_log_file='Relay_Master_Log_File:后的值'，master_log_pos=Exec_Master_Log_Pos：后的值，
	master_auto_position=0;
	
	4.start slave;

## 在Master主库中手工产生数据看是否同步： ##

    1.flush logs；
    2.show master status;
    更改日志格式：
    1.SET @@GLOBAL.GTID_MODE = ON_PERMISSIVE; <---此语句在主从库都执行。
    2.use wubx3306；
    3.show tables;
    4.create table t1 linke zs_user;
    5.show tables;
    6.show master status;
    7.show global variables like "%gtid%";
    8.SET @@GLOBAL.GTID_MODE = OFF_PERMISSIVE;  --从GTID模式转换成非GTID模式
    9.show global variables like "%gtid%";
    10.insert into zs_user (name,add_time) values('wubx',now());
    11.show binary logs;
    12.show binary events in 'mysql-bin.000005'；
    13.select * from zs_user order by uid desc limit 1; --通过倒排序查看最后一次写入的数据记录##主从都可以执行
    14.insert into t1 (name,add_time) values('wubx',now());
	## ***到达slave执行15 16 17 语句*** ##
    18.select @@global.gtid_owned;<--是不是为空
    19.set global gtid_mode=0; 
    20.set global enforce_gtid_consistency=0;


## Slave从库： ##

    15.select * from t1;
    16.show global variables like "%gtid%";
    17.SET @@GLOBAL.GTID_MODE = OFF_PERMISSIVE;修改从库的复制模式。不产生GTID，但是可以处理GTID。
	## master执行18 19 20 语句 ##
    21.set global gtid_mode=0; 
    22.set global enforce_gtid_consistency=0;

----------
# 最后关键把下面语句写入配置文件【mysqld】 #
    [mysqld]
    gtid-mode=OFF
    enforce_gtid_consistency=OFF

----------

# 启用控制：
## 一：master # ##
    1.set @@global.enforce_gtid_consistency = warn;
    2.show master status;
    3.set @@global.enforce_gtid_consistency = on;
    7.SET @@GLOBAL.GTID_MODE = OFF_PERMISSIVE;
    9.show master status;
    14.flush logs;
    15.SET @@GLOBAL.GTID_MODE = ON_PERMISSIVE;<---现在从库执行。
    16.show master status;
    17.show status linke 'ongoing_anonymous_transaction_count';
     value 是否是0
    18.set global gtid_mode=1;

 
## 二：Slave ##
    4.set @@global.enforce_gtid_consistency = warn;
    5.show master status;
    6.set @@global.enforce_gtid_consistency = on;
    8.SET @@GLOBAL.GTID_MODE = OFF_PERMISSIVE;
    10.SET @@GLOBAL.GTID_MODE = ON_PERMISSIVE; <---现在从库执行。
    11.show master status;
    12.show global variable linke "%gtid%";
    13.show slave status;
    19.set global gtid_mode=1;
    20.stop slave;
    21.change masster to master_auto_position=1;
    22.show slave status\G;
    查看Auto_Position:1
    23.start slave;

# 最后关键把下面语句写入配置文件【mysqld】 #
    [mysqld]
    gtid-mode=ON
    enforce_gtid_consistency=ON
