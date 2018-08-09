
  - <h3>前言</h3>

----------- 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;由于最近要使用Mysql数据库，而目前公司服务器与业务有限，于是只使用了一台Mysql。所以，问题很明显，如果这台Mysql坏了，那么将会影响整个公司的业务，所以考虑做Mysql的高可用方案。目前，Mysql的高可用方案很多，这里选择Keepalived+Mysql实现高可用。

<br />

  - <h3>环境介绍</h3>

-----------
 |ID| OS| IP| Role|
 |----| ----- |------ | ------|
 |node1|CentOS6.5_X64 |192.168.1.159 |Master |
 |node2|CentOS6.5_X64|192.168.1.160|Slave |

<br />

 - <h3>环境准备</h3>

-----------

```
# 关闭防火墙(Master和Slave上配置)
[root@node1 ~]# service iptables stop
[root@node1 ~]# chkconfig iptables off
[root@node1 ~]# setenforce 0
[root@node1 ~]# sed -i '/^SELINUX=/{ s/enforcing/disabled/ }' /etc/selinux/config
```

```
# 配置时间同步(Master和Slave上配置)
[root@node1 ~]# yum -y install ntp
[root@node1 ~]# service ntpd start
[root@node1 ~]# \cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
[root@node1 ~]# ntpdate us.pool.ntp.org
```

<br />

 - <h3>Mysql主主配置</h3>

-----------

```
# 在Master和Slave执行
[root@node1 ~]# yum install -y mysql-server mysql      #安装Mysql
[root@node1 ~]# service mysqld start
[root@node1 ~]# mysqladmin -u root password 123.com          #为Mysql的root用户设置密码
[root@node1 ~]# vim /etc/my.cnf         #编辑Mysql配置文件，加入如下内容
[mysqld]
server-id = 1                    #Slave这台设置2
log-bin = mysql-bin
binlog-ignore-db = mysql,information_schema       #忽略写入binlog日志的库
auto-increment-increment = 2             #字段变化增量值
auto-increment-offset = 1              #初始字段ID为1
slave-skip-errors = all

[root@node1 ~]# service mysqld restart
```
<br />
注：先查看Master和Slave上的log bin日志和pos值位置

```
[root@node1 ~]# mysql -u root -p123.com
mysql> show master status;
```
![这里写图片描述](http://img.blog.csdn.net/20160822170214333)
<br /><br />
```
[root@node2 ~]# mysql -u root -p123.com
mysql> show master status;

```

![这里写图片描述](http://img.blog.csdn.net/20160822155022127)



<br />
```
# 注：在Master上操作如下：
[root@node1 ~]# mysql -u root -p123.com
mysql> grant all on *.* to 'root'@'%' identified by '123.com' with grant option;
# with grant option：表示权限传递
mysql> flush privileges;
mysql> change master to
    -> master_host='192.168.1.160',
    -> master_user='root',
    -> master_password='123.com',
    -> master_log_file='mysql-bin.000001',      # 根据对端实际的log bin填写
    -> master_log_pos=106;        # 根据对端实际的Pos值填写
mysql> start master;              # 启动主
```

```
# 注：在Slave上操作如下：
[root@node2 ~]# mysql -u root -p123.com
mysql> grant all on *.* to 'root'@'%' identified by '123.com' with grant option;
mysql> flush privileges;
mysql> change master to
    -> master_host='192.168.1.159',
    -> master_user='root',
    -> master_password='123.com',
    -> master_log_file='mysql-bin.000001',     # 根据对端实际的log bin填写
    -> master_log_pos=335;        # 根据对端实际的Pos值填写
mysql> start slave;               # 启动同步线程
```

```
# 主主同步配置完毕，在Master和Slave上分别查看同步状态，Slave_IO和Slave_SQL是YES说明主主同步成功。
mysql> show slave status\G
```
![这里写图片描述](http://img.blog.csdn.net/20160822170803263)

<br />
 

  - <h3>主主同步测试</h3>

-----------
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;使用客户端连接任意一台Mysql，进行创库、创表、插入字段、删除表、删库等操作，另外一台Mysql都能进行同步。

```
在这里，进行简单一些的测试，先在Master上创建一个新库，名为main
```
![这里写图片描述](http://img.blog.csdn.net/20160822173247952)

![这里写图片描述](http://img.blog.csdn.net/20160822173323515)

```
然后到Slave上查看是否有名为main的库
```
![这里写图片描述](http://img.blog.csdn.net/20160822173430047)

<br />

```
接下来在slave把名为main的库删了，会发现master上的main也没有了。到此，主主同步完成，
```
![这里写图片描述](http://img.blog.csdn.net/20160822173652660)

![这里写图片描述](http://img.blog.csdn.net/20160822173736349)


<br />


 - <h3>Keepalived高可用配置</h3>

-----------

```
注： 在Master和Slave执行，Keepalived必须和Mysql安装在同一台机器上

# 安装Keepalived
[root@node1 ~]# yum install -y pcre-devel openssl-devel popt-devel
[root@node1 ~]# wget http://www.keepalived.org/software/keepalived-1.2.7.tar.gz
[root@node1 ~]# tar zxvf keepalived-1.2.7.tar.gz
[root@node1 ~]# cd keepalived-1.2.7
[root@node1 ~]# ./configure --prefix=/usr/local/keepalived
[root@node1 ~]# make && make install

# 将Keepalived配置成系统服务
[root@node1 ~]# cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/init.d/
[root@node1 ~]# cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
[root@node1 ~]# mkdir /etc/keepalived/
[root@node1 ~]# cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
[root@node1 ~]# cp /usr/local/keepalived/sbin/keepalived /usr/sbin/
```
<br />
```
# 修改Master上的Keepalived配置文件
[root@node1 ~]# vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
     router_id MYSQL_HA
}
 
vrrp_instance VI_1 {
     state BACKUP   # 两台配置此处均是BACKUP
     interface eth0
     virtual_router_id 51
     priority 100   # 优先级，另一台改为90
     advert_int 1
     nopreempt  # 不抢占，只在优先级高的机器上设置即可，优先级低的机器不设置
     
     authentication {
     	auth_type PASS
     	auth_pass 1111
     }
     
     virtual_ipaddress {
     	192.168.1.208     # VIP
     }
}
 
virtual_server 192.168.1.208 3306 {      # VIP的具体设置
    delay_loop 2   # 每个2秒检查一次real_server状态
    #lb_algo wrr   # LVS算法
    #lb_kind DR    # LVS模式
    persistence_timeout 60   # 会话保持时间
    protocol TCP

    real_server 192.168.1.159 3306 {
        weight 3
     	notify_down /usr/local/keepalived/mysql.sh  #检测到服务down后执行的脚本
     	TCP_CHECK {
     	    connect_timeout 10    # 连接超时时间
     	    nb_get_retry 3       # 重连次数
            delay_before_retry 3   # 重连间隔时间
     	    connect_port 3306   # 健康检查端口
     	}
     }
}
```
<br />

```
# 修改Slave上的Keepalived配置文件
[root@node2 ~]# vim /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
     router_id MYSQL_HA
}
 
vrrp_instance VI_1 {
     state BACKUP
     interface eth0
     virtual_router_id 51
     priority 90
     advert_int 1

     authentication {
     	auth_type PASS
     	auth_pass 1111
     }
  
     virtual_ipaddress {
     	192.168.1.208
     }
}
 
virtual_server 192.168.1.208 3306 {
    delay_loop 2
    #lb_algo wrr
    #lb_kind DR
    persistence_timeout 60
    protocol TCP
     
    real_server 192.168.1.160 3306 {
        weight 3
     	notify_down /usr/local/keepalived/mysql.sh
     
	TCP_CHECK {
            connect_timeout 10
            nb_get_retry 3
	    delay_before_retry 3
	    connect_port 3306
     	}
     }
}
```
<br />

```
# 配置故障切换脚本,Master和Slave上执行
[root@node1 ~]# vim /usr/local/keepalived/mysql.sh
#!/bin/sh
pkill keepalived
[root@node1 ~]# chmod +x /usr/local/keepalived/mysql.sh   
```

```
# 此前两台Mysql已经启动，这里只用启动Keepalived(Master和Slave上执行)
[root@node1 ~]# service keepalived start
```
<br />

- <h3>高可用性测试</h3>

-----------


```
# 在Master上查看VIP是否存在
[root@node1 ~]# ip addr
```

![这里写图片描述](http://img.blog.csdn.net/20160823111034794)


----------

```
# 停止Master上的Keepalived，查看VIP是否漂移到了Slave上
[root@node1 ~]# service keepalived stop
[root@node2 ~]# ip addr
```
![这里写图片描述](http://img.blog.csdn.net/20160823111439800)


----------
```
# 开启Master上的Keepalived，查看Master是否抢占了VIP，没有抢占Slave的VIP说明成功
[root@node1 ~]# service keepalived start
```
![这里写图片描述](http://img.blog.csdn.net/20160823111706023)


----------

```
# 停止Slave上的Mysql,发现Slave上Keepalived会自动停止，并且VIP已经漂移到Master上
[root@node2 ~]# service mysqld stop
[root@node2 ~]# ps -ef |grep mysqld
root       4589   2159  0 06:00 pts/2    00:00:00 grep mysqld
[root@node2 ~]# ps -ef |grep keepalived
root       4591   2159  0 06:00 pts/2    00:00:00 grep keepalived
[root@node1 ~]# ip addr
```
![这里写图片描述](http://img.blog.csdn.net/20160823114040381)
<br />

 - <h3>总结</h3>

-----------

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;到此，基于Keepalived实现Mysql高可用全部完成。这个方案可以为Mysql提供容错，缺点也很明显，只有一台Mysql对外提供服务，另外一台只能起到备份的功能，无法实现类似于负载均衡的效果。当然，也可以再加一些中间件完成负载均衡，读写分离等功能，这里就不详解说明了......

