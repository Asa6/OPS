# Mysql主从同步 #

- 环境介绍

| OS            | IP            |  role  |
| ------------- |:-------------:| -----: |
| CentOS6.5_x64 | 192.168.1.232 | Master |
| CentOS6.5_x64 | 192.168.1.208 | Slave  |

</br>

- 环境准备

<pre>
[root@node1 ~]# service iptables stop 
[root@node1 ~]# chkconfig iptables off
[root@node1 ~]# setenforce 0
[root@node1 ~]# sed -i '/^SELINUX=/{ s/enforcing/disabled/ }' /etc/selinux/config
</pre>

</br>

- 主从配置

<pre>
# 修改Master的mysql配置文件
[root@node1 ~]# vim /etc/my.cnf
[mysqld]
log-bin=mysql-bin   # [必须]启用二进制日志
server-id=232       # [必须]服务器唯一ID，默认是1，一般取IP最后一段
[root@node1 ~]# service mysqld restart
</pre>

<pre>
# 修改Slave的mysql配置文件
[root@node2 ~]# vim /etc/my.cnf 
[mysqld]
log-bin=mysql-bin   # [不是必须]启用二进制日志
server-id=208       # [必须]服务器唯一ID，默认是1，一般取IP最后一段
read-only           # 确保从库只读
[root@node2 ~]# service mysqld restart
</pre>

<pre>
# 登录Mysql，在Master上建立帐户并授权slave
# 一般不使用root账户，作为主从账户
# 授权给指定的IP，增加安全性
[root@node1 ~]# mysql -u root -p123.com
mysql> GRANT <font color=#FF0000>REPLICATION SLAVE</font> ON *.* to 'repuser'@'192.168.1.208' identified by '123456';
<font color=#FF0000># REPLICATION SLAVE：用于连接主库进行复制。</font>
mysql> FLUSH PRIVILEGES;     # 权限刷新
</pre>

<pre>
#　在Master上查询Master的状态
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      333 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
# 注：执行完此步骤后不要再操作Master上的Mysql，以防止Master状态值变化
</pre>

<pre>
# 配置Slave
[root@node2 ~]# mysql -u root -p123.com
<font color=#FF0000># change master to: 定义怎样连接Master Mysql</font>
mysql> change master to
    -> master_host='192.168.1.232',
    -> master_user='repuser',
    -> master_password='123456',
    -> master_log_file='mysql-bin.000001',
    -> master_log_pos=333;
mysql> start slave;      # 启动Slave复制功能
</pre>

<pre>
# 检查Slave复制功能状态
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Connecting to master
                  Master_Host: 192.168.1.232   #　Master地址
                  Master_User: repuser    # 授权帐户名，尽量避免使用root
                  Master_Port: 3306   # 数据库端口，部分版本没有此行
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 333 # 同步读取二进制日志的位置，大于等于Exec_Master_Log_Pos的值
               Relay_Log_File: node2-relay-bin.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
			   ......
# 注：Slave_IO及Slave_SQL进程必须正常运行，即YES状态，否则都是错误的状态(如：其中一个NO均属错误)。
</pre>

<p><font color=#FF0000>完成以上操作过程，即主从同步配置完成。</font></p>


- 主从同步测试

<pre>
# 登录主Mysql，建立数据库，在库中建表并插入一条数据
mysql> create database main;
Query OK, 1 row affected (0.07 sec)

mysql> use main;
Database changed
mysql> create table test(id int(4));
Query OK, 0 rows affected (0.46 sec)

mysql> insert into test values(1);
Query OK, 1 row affected (0.07 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| main               |
| mysql              |
| performance_schema |
| test               |
+--------------------+
5 rows in set (0.00 sec)


# 接下来，登录从库，在从Mysql上查询
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| main               |    # 在这里能看到主上新增的库
| mysql              |
| performance_schema |
| test               |
+--------------------+
5 rows in set (0.11 sec)

mysql> use main;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from test;   # 查看主上新增的具体数据
+------+
| id   |
+------+
|    1 |
+------+
1 row in set (0.00 sec)
</pre>

</br>

- 总结
<p><font color=#FF0000>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;可以编写shell脚本，或者配合监控软件（如：Zabbix），监控Slave的两个yes（Slave_IO及Slave_SQL进程），如发现只有一个或零个yes，就表明主从有问题了，发短信警报吧。</font></p>


</br>

- 信息

<p>打开数据库时，出现如下信息：</p>
<p><font color=#FF0000>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A</font></p>


<pre>
出现问题的原因是：
   # 我们进入mysql时，没有使用-A参数，即使用的是以下这种方式进入数据库：
   [root@node1 ~]# mysql -hhostname -uusername -ppassword -Pport
   # 使用-A的方式进入数据库。
   [root@node1 ~]# mysql -hhostname -uusername -ppassword -Pport -A
   # 当打开数据库时，即use main，要预读数据库信息，当使用-A参数时，就不预读数据库信息
   # 当数据库太大，即数据库中表非常多时，预读数据库信息，将非常慢，有可能在打开库的时候卡住；但数据库中表非常少，将不会出现问题。
</pre>

</br>

- Mysql允许远程连接

<pre>
# 新建一个用户用于远程连接，尽量避免使用mysql中的root用户
[root@node1 ~]# GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, ALTER ON *.* TO 'Chanzorpsql'@'%' IDENTIFIED BY "Chanzor123";
</pre>

