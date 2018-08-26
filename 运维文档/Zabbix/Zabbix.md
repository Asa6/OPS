# Zabbix #
作者：HY.Geng<br />
日期：2017年02月27日<br />
版本: zabbix 3.2<br />
官方文档：<a href="http://www.zabbix.com/manuals">http://www.zabbix.com/manuals</a><br />

## Zabbix简介 ##


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Zabbix是一个高度集成的网络监控解决方案，可以提供企业级的开源分布式监控解决方案，由一个国外的团队持续维护更新，软件可以自由下载使用，运作团队靠提供收费的技术支持赢利<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;zabbix是一个基于Web界面的，提供分布式系统监控以及网络监视功能的企业级的开源解决方案。<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;zabbix能监视各种网络参数，保证服务器系统的安全运营，并提供灵活的通知机制以让系统管理员快速定位/解决存在的各种问题<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;zabbix主要由2部分构成zabbix server和zabbix agent，可选组建zabbix proxy<br />
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;zabbix server可以通过SNMP，zabbix agent，fping端口监视等方法对远程服务器或网络状态完成监视，数据收集等功能。同时支持Linux以及Unix平台，Windows平台只能安装客户端

<br />

## Zabbix实战 ##

下列描述均以操作系统 Centos7 为例部署，涉及服务端和客户端两台服务器。

步骤：

- 1 环境介绍
- 2 准备工作
- 3 安装配置 Zabbix-Server
- 4 安装配置 Zabbix-Agent

### 1 环境介绍 ###

| OS            | IP            | HostName      | Role |
| ------------- |:-------------:| ------------- | -------- |
| CentOS7 x64 | 192.168.0.211 | node1         | zabbix-server |
| CentOS7 x64 | 192.168.1.192 | node2         | zabbix-agent |

<br />

### 2 准备工作 ###
<pre>
# 关闭Iptables和SELinux
[root@node1 ~]# systemctl stop firewalld
[root@node1 ~]# systemctl disable firewalld
[root@node1 ~]# setenforce 0
[root@node1 ~]# sed -i '/^SELINUX=/{ s/enforcing/disabled/ }' /etc/selinux/config

# 调整服务器时间
[root@node1 ~]# yum -y install ntp
[root@node1 ~]# ntpdate -u 202.120.2.101
</pre>

<br />

### 3 安装配置 Zabbix-Server ###

步骤：

- 安装配置Mysql
- 安装配置 Zabbix-Server、php、httpd
- Web配置

由于Zabbix-Server需要LAMP环境，所以先安装配置LAMP环境

##### 安装配置mysql5.6 #####

<pre>
# 添加mysql5.6的YUM源
[root@node1 ~]# vim /etc/yum.repos.d/mysql-community.repo
# Enable to use MySQL 5.6
[mysql56-community]
name=MySQL 5.6 Community Server
baseurl=http://repo.mysql.com/yum/mysql-5.6-community/el/6/$basearch/
enabled=1
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql

[root@node1 ~]# yum -y install mysql-community-server.x86_64
[root@node1 ~]# systemctl start mysqld  # 启动mysql
[root@node1 ~]# mysqladmin -u root password '123.com'   # 设置mysql密码
</pre>

<pre>
# 配置数据库
[root@node1 ~]# mysql -u root -p123.com
mysql> create database zabbix character set utf8 collate utf8_bin;
mysql> grant all privileges on zabbix.* to zabbix@localhost identified by 'zabbix';
mysql> quit;
</pre>


##### 安装配置 Zabbix-Server、php、httpd #####

<pre>
# 安装zabbix的Yum源
[root@node1 ~]# rpm -ivh http://repo.zabbix.com/zabbix/3.2/rhel/7/x86_64/zabbix-release-3.2-1.el7.noarch.rpm
</pre>

<pre>
# 安装Zabbix-server及相关依赖（mysql、web）
[root@node1 ~]# yum install zabbix-server-mysql zabbix-web-mysql -y
</pre>

<pre>
# 导入初始数据
[root@node1 ~]# zcat /usr/share/doc/zabbix-server-mysql-3.2.*/create.sql.gz | mysql -uzabbix zabbix -pzabbix
</pre>

<pre>
# 修改zabbix-server 配置文件
[root@node1 ~]# vi /etc/zabbix/zabbix_server.conf
DBHost = localhost
DBName = zabbix
DBUser = zabbix
DBPassword = zabbix

# 启动zabbix-server
[root@node1 ~]# systemctl start zabbix-server
[root@node1 ~]# systemctl enable zabbix-server 
</pre>

<pre>
# 修改zabbix前端的php配置
[root@node1 ~]# vim /etc/httpd/conf.d/zabbix.conf
php_value date.timezone Asia/Shanghai

# 启动http
[root@node1 ~]# systemctl start httpd
[root@node1 ~]# systemctl enable httpd
</pre>


##### Web配置 #####

上面步骤完成后，第一次打开游览器访问http://192.168.0.211/zabbix,会出现如下图：
![](http://i.imgur.com/nm225N8.png)

点击Next step后，如下图：
![](http://i.imgur.com/OZnSbS6.png)

确保所有选项都显示OK，点击Next step后，如下图：
![](http://i.imgur.com/Da2lRSd.png)

确保输入正确的数据库连接信息(默认即可)，点击Next step后,如下图：
![](http://i.imgur.com/UMq4GHz.png)

保持默认即可，点击Next step后，如下图：
![](http://i.imgur.com/nUBmJiq.png)

确认无误，点击Next step后，如下图：
![](http://i.imgur.com/ZdrbXZI.png)

点击Finish后，如下图：
![](http://i.imgur.com/WXYWcU9.png)


现在zabbix的所有安装配置完成，只要输入用户名和密码即可登录Web<br />
默认用户名: Admin<br />
默认密码: zabbix

<br />

### 4 安装配置Zabbix-Agent ###

- 1 准备工作
- 2 安装配置 Zabbix-Agent
- 3 在Web上配置监听

现在使用zabbix监控主机，为了方便测试就不监控多台了，一台即可

##### 准备工作 #####
<pre>
# 所以关闭Iptables和SELinux
[root@node2 ~]# systemctl stop firewalld
[root@node2 ~]# systemctl disable firewalld
[root@node2 ~]# setenforce 0
[root@node2 ~]# sed -i '/^SELINUX=/{ s/enforcing/disabled/ }' /etc/selinux/config

# 调整服务器时间
[root@node2 ~]# yum -y install ntp
[root@node2 ~]# ntpdate -u 202.120.2.101
</pre>

##### 安装配置zabbix-agent #####

<pre>
# 安装zabbix的yum源
[root@node2 ~]# rpm -ivh http://repo.zabbix.com/zabbix/3.2/rhel/7/x86_64/zabbix-release-3.2-1.el7.noarch.rpm

# 安装zabbix-agent
[root@node2 ~]# yum -y install zabbix-agent
</pre>

<pre>
# 修改zabbix-agent的配置文件
[root@node2 ~]# vim /etc/zabbix/zabbix_agentd.conf
Server=192.168.0.211   # 填写Zabbix-server的地址
ServerActive=192.168.0.211   # 填写Zabbix-server的地址
Hostname=client1
</pre>

<pre>
[root@node2 ~]# systemctl start zabbix-agent
[root@node2 ~]# systemctl enable zabbix-agent
</pre>

##### 在Web上配置监听 #####

完成上述配置后，打开http://192.168.0.211/zabbix,输入用户名、密码登录，点击Configuration下的Hosts，如下图：
![](http://i.imgur.com/Xs53HLD.png)

点击Create host，填写相关信息，如下图：
![](http://i.imgur.com/C1o1LHW.png)

接着点击Templates,这里选择名为Template OS Linux的监控模板并添加，如下图：
![](http://i.imgur.com/BszrtmJ.png)

最后点击ADD，如下图：
![](http://i.imgur.com/ToNWsMI.png)

等待成功，如下图：
![](http://i.imgur.com/GhTlx07.png)

最后，点击Monitoring下的Graphs，选择正确的Group、Host、Graphs，即可看到对应的监控图表，如下图：
![](http://i.imgur.com/yJnaDZB.png)


<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;到此，全部结束。不过zabbix自带的监控图表有点不好看，可以选择Grafana（拥有更加绚丽显示）配合zabbix使用；并且，一些更加细致的监控和功能也没有详细说明，以后再更加深入使用。
