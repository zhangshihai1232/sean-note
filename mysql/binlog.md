## mysql5.7配置文件
ubuntu下apt-get安装的mysql5.7配置路径如下

```bash
/etc/mysql/mysql.conf.d/mysqld.cnf
```

## binlog日志
记录了DDL和DML,事件形式记录，还包含语句所执行的消耗的时间，MySQL的二进制日志是事务安全型的
两个使用场景：

```bash
其一：MySQL Replication在Master端开启binlog，Mster把它的二进制日志传递给slaves来达到master-slave数据一致的目的
其二：数据恢复了，mysqlbinlog工具
```
包括两类文件：
1. 日志索引文件（后缀.index记录所有的二进制文件）
2. 二进制日志文件(后缀.00000*)记录DDL和DML事件

## binlog管理
常用操作

```bash
## 查看状态
show variables like '%log_bin%';
## 查看master状态
show master status;
## 刷新log日志，产生一个新编号的binlog日志文件
## 每当mysqld服务重启时，会自动执行此命令，刷新binlog日志；在mysqldump备份数据时加 -F 选项也会刷新binlog日志；
flush logs;
## 重置(清空)所有binlog日志
reset master
```

开启
修改`my.cnf`把这一样的注释去掉`一定要开启server-id`

```bash
server-id		= 1
log_bin			= /var/log/mysql/mysql-bin.log
```
然后重启mysql
注意：

```bash
1. bin-log的目录所有者需要为mysql
2. bin-log的存储磁盘需要大于3G
3. server-id需要开启
```
max_binlog_size设置bin-log文件大小
binlog-cache-size=100m二进制日志缓存大小
sync-binlog=N每隔N秒将缓存日志写入磁盘，在replication环境中一般设置为1，
还需要设置innodb_flush_log_at_trx_commit=1以及innodb-support-xa=1默认开启

![](http://www.linuxidc.com/upload/2014_09/140923193363491.png)


## 查看方式
mysqlbinlog工具

```bash
mysqlbinlog /data/mysql/mysql-bin.000001
```
查询的结果不好阅读

```bash
# at 1188
#161109 23:23:03 server id 1  end_log_pos 1307 CRC32 0x28a820c9         Query   thread_id=4     exec_time=0     error_code=0
use `test`/*!*/;
SET TIMESTAMP=1478704983/*!*/;
alter table books add price float default 1.1
/*!*/;
# at 1307
#161109 23:30:50 server id 1  end_log_pos 1372 CRC32 0xa50a65d1         Anonymous_GTID  last_committed=4        sequence_number=5
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 1372
#161109 23:30:50 server id 1  end_log_pos 1492 CRC32 0x54e88245         Query   thread_id=4     exec_time=0     error_code=0
SET TIMESTAMP=1478705450/*!*/;
alter table books add price2 float default 1.1
/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
```
在mysql中的查询命令

```bash
show binlog events [IN 'log_name'] [FROM pos] [LIMIT [offset,] row_count];
```
IN 'log_name'   指定要查询的binlog文件名(不指定就是第一个binlog文件)
FROM pos        指定从哪个pos起始点开始查起(不指定就是从整个文件首个pos点开始算)
LIMIT [offset,] 偏移量(不指定就是0)
row_count       查询总条数(不指定就是所有行)
结果如下

```bash
+------------------+------+----------------+-----------+-------------+------------------------------------------------------------+
| Log_name         | Pos  | Event_type     | Server_id | End_log_pos | Info                                                       |
+------------------+------+----------------+-----------+-------------+------------------------------------------------------------+
| mysql-bin.000002 |    4 | Format_desc    |         1 |         123 | Server ver: 5.7.11-0ubuntu6-log, Binlog ver: 4             |
| mysql-bin.000002 |  123 | Previous_gtids |         1 |         154 |                                                            |
| mysql-bin.000002 |  154 | Anonymous_Gtid |         1 |         219 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                       |
| mysql-bin.000002 |  219 | Query          |         1 |         291 | BEGIN                                                      |
| mysql-bin.000002 |  291 | Table_map      |         1 |         354 | table_id: 108 (test.books)                                 |
| mysql-bin.000002 |  354 | Delete_rows    |         1 |         443 | table_id: 108 flags: STMT_END_F                            |
| mysql-bin.000002 |  443 | Xid            |         1 |         474 | COMMIT /* xid=12 */                                        |
| mysql-bin.000002 |  474 | Anonymous_Gtid |         1 |         539 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                       |
| mysql-bin.000002 |  539 | Query          |         1 |         611 | BEGIN                                                      |
| mysql-bin.000002 |  611 | Table_map      |         1 |         674 | table_id: 108 (test.books)                                 |
| mysql-bin.000002 |  674 | Delete_rows    |         1 |         765 | table_id: 108 flags: STMT_END_F                            |
| mysql-bin.000002 |  765 | Xid            |         1 |         796 | COMMIT /* xid=13 */                                        |
| mysql-bin.000002 |  796 | Anonymous_Gtid |         1 |         861 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                       |
| mysql-bin.000002 |  861 | Query          |         1 |         941 | BEGIN                                                      |
| mysql-bin.000002 |  941 | Table_map      |         1 |        1004 | table_id: 108 (test.books)                                 |
| mysql-bin.000002 | 1004 | Write_rows     |         1 |        1092 | table_id: 108 flags: STMT_END_F                            |
| mysql-bin.000002 | 1092 | Xid            |         1 |        1123 | COMMIT /* xid=15 */                                        |
| mysql-bin.000002 | 1123 | Anonymous_Gtid |         1 |        1188 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                       |
| mysql-bin.000002 | 1188 | Query          |         1 |        1307 | use `test`; alter table books add price float default 1.1  |
| mysql-bin.000002 | 1307 | Anonymous_Gtid |         1 |        1372 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                       |
| mysql-bin.000002 | 1372 | Query          |         1 |        1492 | use `test`; alter table books add price2 float default 1.1 |
+------------------+------+----------------+-----------+-------------+------------------------------------------------------------+
```

```bash
##查询第一个binlog日志
show binlog events\G;
#指定查询 mysql-bin.000021 这个文件
show binlog events in 'mysql-bin.000021'\G;
#指定查询 mysql-bin.000021 这个文件，从pos点:8224开始查起：
show binlog events in 'mysql-bin.000021' from 8224\G;
#指定查询 mysql-bin.000021 这个文件，从pos点:8224开始查起，查询10条
show binlog events in 'mysql-bin.000021' from 8224 limit 10\G;
#指定查询 mysql-bin.000021 这个文件，从pos点:8224开始查起，偏移2行，查询10条
show binlog events in 'mysql-bin.000021' from 8224 limit 2,10\G;
```

## 恢复binlog日志
假设4:00执行了备份，使用了-F将`zyyshop`数据库备份到`/root/BAK.zyyshop.sql`中
这时sql中查询Log只显示同步之后的增删改

```bash
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```
下午18:00执行了`drop database zyyshop`，整个 数据库没了
这时查询binlog日志，找到关键的pos点，使用` flush logs;`刷新，开启新的日志
推荐使用利用mysql查看的方式`show binlog events in 'mysql-bin.000023';`恢复pos之前的即可
先恢复备份数据

```bash
/usr/local/mysql/bin/mysql -uroot -p123456 -v < /root/BAK.zyyshop.sql;
```
再利用binlog恢复其他数据

```bash
 mysqlbinlog mysql-bin.0000xx | mysql -u用户名 -p密码 数据库名
```
常用选项：
- start-position=953                   起始pos点
- stop-position=1437                   结束pos点
- start-datetime="2013-11-29 13:18:54" 起始时间点
- stop-datetime="2013-11-29 13:21:53"  结束时间点
--database=zyyshop                     指定只恢复zyyshop数据库(一台主机上往往有多个数据库，只限本地log日志)

不常用选项：    
-u --user=name              Connect to the remote server as username.连接到远程主机的用户名
-p --password[=name]        Password to connect to remote server.连接到远程主机的密码
-h --host=name              Get the binlog from server.从远程主机上获取binlog日志
--read-from-remote-server   Read binary logs from a MySQL server.从某个MySQL服务器上读取binlog日志
## 恢复方式
**全部恢复**
因为这个例子中，有一条`drop database zyyshop`需要去掉，然后执行

```bash
/usr/local/mysql/bin/mysqlbinlog  /usr/local/mysql/data/mysql-bin.000021 | /usr/local/mysql/bin/mysql -uroot -p123456 -v zyyshop 
```
**指定pos结束点恢复**
@ --stop-position=953 pos结束点
此pos结束点介于“导入实验数据”与更新“name='李四'”之间，这样可以恢复到更改“name='李四'”之前的“导入测试数据”

```bash
/usr/local/mysql/bin/mysqlbinlog --stop-position=953 --database=zyyshop /usr/local/mysql/data/mysql-bin.000023 | /usr/local/mysql/
```

**指定pos点区间恢复**
更新 name='李四' 这条数据，日志区间是Pos[1038] --> End_log_pos[1164]，按事务区间是：Pos[953] --> End_log_pos[1195]；
更新 name='小二' 这条数据，日志区间是Pos[1280] --> End_log_pos[1406]，按事务区间是：Pos[1195] --> End_log_pos[1437]；
单独恢复 name='李四' 这步操作

```bash
 /usr/local/mysql/bin/mysqlbinlog --start-position=1280 --stop-position=1406 --database=zyyshop  /usr/local/mysql/data/mysql-bin.000023 | /usr/local/mysql/bin/mysql -uroot -p123456 -v zyyshop
```
按事务区间单独恢复

```bash
 /usr/local/mysql/bin/mysqlbinlog --start-position=953 --stop-position=1195 --database=zyyshop  /usr/local/mysql/data/mysql-bin.000023 | /usr/local/mysql/bin/mysql -uroot -p123456 -v zyyshop
```
单独恢复 name='小二' 这步操作

```bash
 /usr/local/mysql/bin/mysqlbinlog --start-position=1280 --stop-position=1406 --database=zyyshop  /usr/local/mysql/data/mysql-bin.000023 | /usr/local/mysql/bin/mysql -uroot -p123456 -v zyyshop
```
按事务区间单独恢复

```bash
/usr/local/mysql/bin/mysqlbinlog --start-position=1195 --stop-position=1437 --database=zyyshop  /usr/local/mysql/data/mysql-bin.000023 | /usr/local/mysql/bin/mysql -uroot -p123456 -v zyyshop
```
将 name='李四'、name='小二' 多步操作一起恢复，需要按事务区间

```bash
/usr/local/mysql/bin/mysqlbinlog --start-position=953 --stop-position=1437 --database=zyyshop  /usr/local/mysql/data/mysql-bin.000023 | /usr/local/mysql/bin/mysql -uroot -p123456 -v zyyshop
```

**指定时间区间恢复**
需要用mysqlbinlog命令读取binlog日志内容，找时间节点
@ --start-datetime="2013-11-29 13:18:54"  起始时间点
@ --stop-datetime="2013-11-29 13:21:53"   结束时间点

```bash
/usr/local/mysql/bin/mysqlbinlog --start-datetime="2013-11-29 13:18:54" --stop-datetime="2013-11-29 13:21:53" --database=zyyshop /usr/local/mysql/data/mysql-bin.000021 | /usr/local/mysql/bin/mysql -uroot -p123456 -v zyyshop
```
所谓恢复，就是让mysql将保存在binlog日志中指定段落区间的sql语句逐个重新执行一次而已