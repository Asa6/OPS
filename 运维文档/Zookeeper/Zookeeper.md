# Zookeeper集群部署实战 #
作者：HY.Geng<br />
日期：2016年11月23日


## Zookeeper简介 ##

<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;zookeeper是一个分布式的开源框架，它能很好的管理集群，而且提供协调分布式应用的基本服务。它向外部应用暴露一组通用服务——分布式同步（Distributed Synchronization）、命名服务（Naming Service）、集群维护（GroupMaintenance）等，简化分布式应用协调及其管理的难度，提供高性能的分布式服务。zookeeper本身可以以standalone模式（单节点状态）安装运行，不过它的长处在于通过分布式zookeeper集群（一个leader，多个follower），基于一定的策略来保证zookeepe。</p>

<br />

## Zookeeper集群角色介绍 ##
 
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;zookeeper集群中主要有两个角色：leader和follower。</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;领导者（leader）,用于负责进行投票的发起和决议,更新系统状态。</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;学习者（learner）,包括跟随者（follower）和观察者（observer）。</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;其中follower用于接受客户端请求并想客户端返回结果,在选主过程中参与投票。而observer可以接受客户端连接,将写请求转发给leader,但observer不参加投票过程,只同步leader的状态,observer的目的是为了扩展系统,提高读取速度。</p>

<br />

## Zookeeper集群节点个数 ##
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;一个zookeeper集群需要运行几个zookeeper节点呢？你可以运行一个zookeeper节点，但那就不是集群了。如果要运行zookeeper集群的话，最好部署3，5，7个zookeeper节点。本次实验我们是以3个节点进行的。zookeeper节点部署的越多，服务的可靠性也就越高。当然建议最好是部署奇数个，偶数个不是不可以。但是zookeeper集群是以宕机个数过半才会让整个集群宕机的，所以奇数个集群更佳。你需要给每个zookeeper 1G左右的内存，如果可能的话，最好有独立的磁盘，因为独立磁盘可以确保zookeeper是高性能的。如果你的集群负载很重，不要把zookeeper和RegionServer运行在同一台机器上面，就像DataNodes和TaskTrackers一样。</p>

<br />

## 环境介绍 ##


| OS            | IP            | HostName      |
| ------------- |:-------------:| ------------- |
| CentOS6.5_x64 | 192.168.1.232 | node1         |
| CentOS6.5_x64 | 192.168.1.208 | node2         |
| CentOS6.5_x64 | 192.168.1.237 | node3         |

<br />

## 环境准备 ##

<pre>
# 关闭防火墙，所有节点上执行
[root@node1 ~]# service iptables stop 
[root@node1 ~]# chkconfig iptables off
[root@node1 ~]# setenforce 0
[root@node1 ~]# sed -i '/^SELINUX=/{ s/enforcing/disabled/ }' /etc/selinux/config
</pre>

<br />

## Zookeeper安装 ##

<pre>
# 由于Zookeepr运行需要JAVA环境，所以3台机器全都安装JDK环境
[root@node1 ~]# yum -y install java-1.8.0-openjdk*
[root@node1 ~]# java -version
openjdk version "1.8.0_111"
OpenJDK Runtime Environment (build 1.8.0_111-b15)
OpenJDK 64-Bit Server VM (build 25.111-b15, mixed mode)
</pre>

<pre>
# 安装ZK， 3台机器全都安装
[root@node1 ~]# wget http://mirrors.cnnic.cn/apache/zookeeper/zookeeper-3.4.8/zookeeper-3.4.8.tar.gz
[root@node1 ~]# tar zxvf zookeeper-3.4.8.tar.gz -C /usr/local/
[root@node1 ~]# cd /usr/local/zookeeper-3.4.8/
[root@node1 ~]# cp conf/zoo_sample.cfg conf/zoo.cfg
[root@node1 ~]# vim /etc/profile    # 加入环境变量
export PATH=$PATH:/usr/local/zookeeper-3.4.8/bin
[root@node1 ~]# source /etc/profile
[root@node1 ~]# env   # 查看环境变量是否添加成功
HOSTNAME=node1
SELINUX_ROLE_REQUESTED=
TERM=xterm
SHELL=/bin/bash
HISTSIZE=1000
SSH_CLIENT=192.168.0.102 53534 22
SELINUX_USE_CURRENT_RANGE=
OLDPWD=/root
SSH_TTY=/dev/pts/0
USER=root
LS_COLORS=rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=01;05;37;41:su=37;41:sg=30;43:ca=30;41:tw=30;42:ow=34;42:st=37;44:ex=01;32:*.tar=01;31:*.tgz=01;31:*.arj=01;31:*.taz=01;31:*.lzh=01;31:*.lzma=01;31:*.tlz=01;31:*.txz=01;31:*.zip=01;31:*.z=01;31:*.Z=01;31:*.dz=01;31:*.gz=01;31:*.lz=01;31:*.xz=01;31:*.bz2=01;31:*.tbz=01;31:*.tbz2=01;31:*.bz=01;31:*.tz=01;31:*.deb=01;31:*.rpm=01;31:*.jar=01;31:*.rar=01;31:*.ace=01;31:*.zoo=01;31:*.cpio=01;31:*.7z=01;31:*.rz=01;31:*.jpg=01;35:*.jpeg=01;35:*.gif=01;35:*.bmp=01;35:*.pbm=01;35:*.pgm=01;35:*.ppm=01;35:*.tga=01;35:*.xbm=01;35:*.xpm=01;35:*.tif=01;35:*.tiff=01;35:*.png=01;35:*.svg=01;35:*.svgz=01;35:*.mng=01;35:*.pcx=01;35:*.mov=01;35:*.mpg=01;35:*.mpeg=01;35:*.m2v=01;35:*.mkv=01;35:*.ogm=01;35:*.mp4=01;35:*.m4v=01;35:*.mp4v=01;35:*.vob=01;35:*.qt=01;35:*.nuv=01;35:*.wmv=01;35:*.asf=01;35:*.rm=01;35:*.rmvb=01;35:*.flc=01;35:*.avi=01;35:*.fli=01;35:*.flv=01;35:*.gl=01;35:*.dl=01;35:*.xcf=01;35:*.xwd=01;35:*.yuv=01;35:*.cgm=01;35:*.emf=01;35:*.axv=01;35:*.anx=01;35:*.ogv=01;35:*.ogx=01;35:*.aac=01;36:*.au=01;36:*.flac=01;36:*.mid=01;36:*.midi=01;36:*.mka=01;36:*.mp3=01;36:*.mpc=01;36:*.ogg=01;36:*.ra=01;36:*.wav=01;36:*.axa=01;36:*.oga=01;36:*.spx=01;36:*.xspf=01;36:
MAIL=/var/spool/mail/root
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/root/bin:/usr/local/zookeeper-3.4.8/bin
PWD=/usr/local/zookeeper-3.4.8
LANG=en_US.UTF-8
SELINUX_LEVEL_REQUESTED=
HISTCONTROL=ignoredups
SHLVL=1
HOME=/root
LOGNAME=root
SSH_CONNECTION=192.168.0.102 53534 192.168.1.232 22
LESSOPEN=|/usr/bin/lesspipe.sh %s
G_BROKEN_FILENAMES=1
_=/bin/env
</pre>

<br />

## Zookeeper集群搭建 ##

<pre>
# 所有ZK机器的配置一样，如果是一台机器，需要注意端口的配置
[root@node1 ~]# vim /usr/local/zookeeper-3.4.8/conf/zoo.cfg   # 修改配置文件如下
tickTime=2000
initLimit=10
syncLimit=5
clientPort=2181
dataLogDir=/usr/local/zookeeper-3.4.8/logs
dataDir=/usr/local/zookeeper-3.4.8/data
server.1= 192.168.1.232:2888:3888
server.2= 192.168.1.208:2888:3888
server.3= 192.168.1.237:2888:3888

# 配置文件参数说明:

#	tickTime这个时间是作为zookeeper服务器之间或客户端与服务器之间维持心跳的时间间隔,也就是说每个tickTime时间就会发送一个心跳。

#	initLimit这个配置项是用来配置zookeeper接受客户端（这里所说的客户端不是用户连接zookeeper服务器的客户端,而是zookeeper服务器集群中连接到leader的follower 服务器）初始化连接时最长能忍受多少个心跳时间间隔数。
#	当已经超过10个心跳的时间（也就是tickTime）长度后 zookeeper 服务器还没有收到客户端的返回信息,那么表明这个客户端连接失败。总的时间长度就是 10*2000=20秒。

#	syncLimit这个配置项标识leader与follower之间发送消息,请求和应答时间长度,最长不能超过多少个tickTime的时间长度,总的时间长度就是5*2000=10秒。

#	dataDir顾名思义就是zookeeper保存数据的目录,默认情况下zookeeper将写数据的日志文件也保存在这个目录里；

#	clientPort这个端口就是客户端连接Zookeeper服务器的端口,Zookeeper会监听这个端口接受客户端的访问请求；

#	server.A=B:C:D中的A是一个数字,表示这个是第几号服务器,B是这个服务器的IP地址，C第一个端口用来集群成员的信息交换,表示这个服务器与集群中的leader服务器交换信息的端口，D是在leader挂掉时专门用来进行选举leader所用的端口。


# 在所有机器上创建Zookeeper数据目录和日志目录
[root@node1 ~]# mkdir -p /usr/local/zookeeper-3.4.8/logs
[root@node1 ~]# mkdir -p /usr/local/zookeeper-3.4.8/data
</pre>

<pre>
# 除了修改zoo.cfg配置文件外,zookeeper集群模式下还要配置一个myid文件,这个文件需要放在dataDir目录下。
# 这个文件里面有一个数据就是A的值（该A就是zoo.cfg文件中server.A=B:C:D中的A）,在zoo.cfg文件中配置的dataDir路径中创建myid文件。

# 在192.168.1.232服务器上创建myid文件，并设置为1，同时与zoo.cfg文件里面的server.1对应，如下：
[root@node1 ~]# echo "1" > /usr/local/zookeeper-3.4.8/data/myid

# 在192.168.1.208服务器上创建myid文件，并设置为2，同时与zoo.cfg文件里面的server.2对应，如下：
[root@node2 ~]# echo "2" > /usr/local/zookeeper-3.4.8/data/myid

# 在192.168.1.237服务器上创建myid文件，并设置为3，同时与zoo.cfg文件里面的server.3对应，如下：
[root@node3 ~]# echo "3" > /usr/local/zookeeper-3.4.8/data/myid
</pre>

<br />

## Zookeeper集群启动 ##
<pre>
# 所有ZK机器上执行
# 注意:在启动第一台zookeeper的时候可能会报错，等三台zookeeper全部启动完成之后就不会报错了。
[root@node1 ~]# zkServer.sh start   # 启动
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper-3.4.8/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
</pre>

<br />

## Zookeeper集群查看 ##

<pre>
# 在192.168.1.232上查看
[root@node1 ~]# zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper-3.4.8/bin/../conf/zoo.cfg
Mode: follower

# 在192.168.1.208上查看
[root@node2 ~]# zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper-3.4.8/bin/../conf/zoo.cfg
Mode: follower

# 在192.168.1.237上查看
[root@node3 ~]# zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /usr/local/zookeeper-3.4.8/bin/../conf/zoo.cfg
Mode: leader

# 我们可以很明显的看出192.168.1.232和192.168.1.208这两台服务器zookeeper的状态是follow模式，192.168.1.237这台服务器zookeeper的状态是leader模式。
# 这说明zookeeper集群已经成功搭建。
</pre>

<br />

## 连接zookeeper集群 ##

<pre>
# zookeeper集群搭建完毕后，我们可以通过客户端脚本，连接到zookeeper集群上。
# 对于客户端来说，zookeeper集群是一个整体，连接到zookeeper集群实际上感觉在独享整个集群的服务，所以，你可以在任何一个结点上建立到服务集群的连接，例如：
[root@node1 ~]# zkCli.sh -server 192.168.1.208:2181   # 连接ZK集群
[zk: 192.168.1.208:2181(CONNECTED) 0] ls /
[zookeeper]
# 我们可以很明显的看出在192.168.1.232这台机器上连接192.168.1.208服务器上的zookeeper是正常的，而且当前根路径为/zookeeper。
# 到此有关zookeeper集群搭建就完全结束。
</pre>
