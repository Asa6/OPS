# Mysql编译安装 #

- 环境介绍

| OS            | IP     
| ------------- | ------------
| CentOS6.5_x64 | 192.168.1.232 

</br>

- 安装Mysql
<pre>
[root@node1 ~]# yum list installed | grep mysql*   # 查询是否已经安装过Mysql
mysql-libs.x86_64       5.1.71-1.el6    @anaconda-CentOS-201311272149.x86_64/6.5
[root@node1 ~]# yum -y remove mysql-libs.x86_64    # 卸载原先的Mysql

[root@node1 ~]# yum -y install gcc gcc-c++ ncurses-devel perl  # 安装依赖环境
[root@node1 ~]# wget http://www.cmake.org/files/v2.8/cmake-2.8.10.2.tar.gz  # 安装cmake
[root@node1 ~]# tar -zxvf cmake-2.8.10.2.tar.gz
[root@node1 ~]# cd cmake-2.8.10.2
[root@node1 cmake-2.8.10.2]# ./bootstrap
[root@node1 cmake-2.8.10.2]# make && make install
[root@node1 cmake-2.8.10.2]# cd

[root@node1 ~]# groupadd mysql               # 新建Mysql组
[root@node1 ~]# useradd -r -g mysql mysql    # 新建Mysql用户
[root@node1 ~]# mkdir -p /opt/mysql          # 新建mysql安装目录
[root@node1 ~]# mkdir -p /data/mysqldb       # 新建mysql数据库数据文件目录

[root@node1 ~]# tar zxvf mysql-5.6.32.tar.gz  # 去mysql官方下载安装包：http://www.mysql.com/
[root@node1 ~]# cd mysql-5.6.32
[root@node1 ~]# cmake \
                      -DCMAKE_INSTALL_PREFIX=/opt/mysql \
                      -DMYSQL_UNIX_ADDR=/opt/mysql/mysql.sock \
                      -DDEFAULT_CHARSET=utf8 \
                      -DDEFAULT_COLLATION=utf8_general_ci \
                      -DWITH_INNOBASE_STORAGE_ENGINE=1 \
                      -DWITH_ARCHIVE_STORAGE_ENGINE=1 \
                      -DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
                      -DMYSQL_DATADIR=/data/mysqldb \
                      -DMYSQL_TCP_PORT=3306 \
                      -DENABLE_DOWNLOADS=1
</pre>
注释：</br>
<p><font color=#FF0000>-DCMAKE_INSTALL_PREFIX=dir_name</font>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp&nbsp;&nbsp;&nbsp;设置mysql安装目录</p>
<p><font color=#FF0000>-DMYSQL_UNIX_ADDR=file_name</font>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp&nbsp;&nbsp;&nbsp;设置监听套接字路径，这必须是一个绝对路径名。默认为/tmp/mysql.sock</p>
<p><font color=#FF0000>-DDEFAULT_CHARSET=charset_name</font>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;设置服务器的字符集。缺省情况下，MySQL使用latin1的（CP1252西欧）字符集。</p>
<p><font color=#FF0000>-DDEFAULT_COLLATION=collation_name</font>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;设置服务器的排序规则。</p>
<p><font color=#FF0000>-DWITH_INNOBASE_STORAGE_ENGINE=1</font>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;支持InnoDB存储引擎</p>
<p><font color=#FF0000>-DWITH_ARCHIVE_STORAGE_ENGINE=1</font>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;支持archive存储引擎</p>
<p><font color=#FF0000>-DWITH_BLACKHOLE_STORAGE_ENGINE=1</font>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp&nbsp;&nbsp;支持blackhole存储引擎</p>
<p><font color=#FF0000>-DWITH_PERFSCHEMA_STORAGE_ENGINE=1</font>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;支持PERFSCHEMA存储引擎</p>
<p><font color=#FF0000>可用的存储引擎值有：</font>ARCHIVE, BLACKHOLE, EXAMPLE, FEDERATED, INNOBASE (InnoDB), PARTITION (partitioning support), 和PERFSCHEMA (Performance Schema)</p>
<p><font color=#FF0000>-DMYSQL_DATADIR=dir_name</font>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;设置mysql数据库文件目录</p>
<p><font color=#FF0000>-DMYSQL_TCP_PORT=port_num	</font>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;设置mysql服务器监听端口，默认为3306</p>
<p><font color=#FF0000>-DENABLE_DOWNLOADS=bool</font>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;是否要下载可选的文件。例如，启用此选项（设置为1），cmake将下载谷歌所使用的测试套件运行单元测试。</p>

</br>

<pre>
[root@node1 ~]# make && make install

# 修改mysql目录所有者和组
[root@node1 ~]# cd /opt/mysql
[root@node1 mysql]# chown -R mysql:mysql .
[root@node1 mysql]# cd /data/mysqldb
[root@node1 mysqldb]# chown -R mysql:mysql .
[root@node1 mysqldb]# cd /opt/mysql

# 初始化mysql数据库
[root@node1 mysql]# scripts/mysql_install_db --user=mysql --datadir=/data/mysqldb

# 复制mysql服务启动配置文件
[root@node1 mysql]# cp /opt/mysql/support-files/my-default.cnf /etc/my.cnf

# 复制mysql服务启动脚本及加入PATH路径
[root@node1 mysql]# cp support-files/mysql.server /etc/init.d/mysqld
[root@node1 mysql]# vim /etc/profile
	PATH=/opt/mysql/bin:/opt/mysql/lib:$PATH
	export PATH
[root@node1 mysql]# source /etc/profile  # 立即生效

# 启动mysql服务并加入开机自启动
[root@node1 mysql]# service mysqld start   
[root@node1 mysql]# chkconfig --level 35 mysqld on
[root@node1 mysql]# cd
</pre>

<pre>
# 检查mysql服务是否启动
[root@node1 ~]# netstat -napt |grep 3306
tcp        0      0 :::3306                     :::*                        LISTEN      1100/mysqld
[root@node1 ~]# mysql -u root -p
# 默认密码为空，如果能登陆上，则安装成功。

# 修改MySQL用户root的密码
[root@node1 ~]# mysqladmin -u root password '123.com'
</pre>