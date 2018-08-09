# codis集群部署实战 #
作者：耿浩源<br />
日期：2016年11月23日
<br />

## Codis简介 ##

<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Codis 是一个分布式 Redis 解决方案, 对于上层的应用来说, 连接到 Codis Proxy 和连接原生的 Redis Server 没有显著区别 (不支持的命令列表), 上层应用可以像使用单机的 Redis 一样使用, Codis 底层会处理请求的转发, 不停机的数据迁移等工作, 所有后边的一切事情, 对于前面的客户端来说是透明的, 可以简单的认为后边连接的是一个内存无限大的 Redis 服务。</p>

## Codis架构图 ##

![](http://i.imgur.com/VxvNE6h.png)

## Codis组件介绍 ##


<p>Codis 3.x 由以下组件组成：</p>


&nbsp;&nbsp;&nbsp;●&nbsp;&nbsp;**Codis Server：**基于 redis-2.8.21 分支开发。增加了额外的数据结构，以支持 slot 有关的操作以及数据迁移指令。具体的修改可以参考文档 <a href="https://github.com/CodisLabs/codis/blob/release3.1/doc/redis_change_zh.md">redis 的修改</a>。

&nbsp;&nbsp;&nbsp;●&nbsp;&nbsp;**Codis Proxy：**客户端连接的 Redis 代理服务, 实现了 Redis 协议。 除部分命令不支持以外(<a href="https://github.com/CodisLabs/codis/blob/release3.1/doc/unsupported_cmds.md">不支持的命令列表</a>)，表现的和原生的 Redis 没有区别（就像 Twemproxy）。

<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;◎ 对于同一个业务集群而言，可以同时部署多个 codis-proxy 实例；</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;◎ 不同 codis-proxy 之间由 codis-dashboard 保证状态同步。</p>

&nbsp;&nbsp;&nbsp;●&nbsp;&nbsp;**Codis Dashboard：**集群管理工具，支持 codis-proxy、codis-server 的添加、删除，以及据迁移等操作。在集群状态发生改变时，codis-dashboard 维护集群下所有 codis-proxy 的状态的一致性。

<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;◎ 对于同一个业务集群而言，同一个时刻 codis-dashboard 只能有 0个或者1个；</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;◎ 所有对集群的修改都必须通过 codis-dashboard 完成。</p>
&nbsp;&nbsp;&nbsp;●&nbsp;&nbsp;**Codis Admin：**集群管理的命令行工具。

<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;◎ 可用于控制 codis-proxy、codis-dashboard 状态以及访问外部存储。</p>
&nbsp;&nbsp;&nbsp;●&nbsp;&nbsp;**Codis FE：**集群管理界面。

<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;◎ 多个集群实例共享可以共享同一个前端展示页面；</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;◎ 通过配置文件管理后端 codis-dashboard 列表，配置文件可自动更新。</p>
&nbsp;&nbsp;&nbsp;●&nbsp;&nbsp;**Codis HA：**为集群提供高可用。

<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;◎ 依赖 codis-dashboard 实例，自动抓取集群各个组件的状态；</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;◎ 会根据当前集群状态自动生成主从切换策略，并在需要时通过 codis-dashboard 完成主从切换。</p>
&nbsp;&nbsp;&nbsp;●&nbsp;&nbsp;Storage：为集群状态提供外部存储。

<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;◎ 提供 Namespace 概念，不同集群的会按照不同 product name 进行组织；</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;◎ 目前仅提供了 Zookeeper 和 Etcd 两种实现，但是提供了抽象的 interface 可自行扩展。</p>

## 环境介绍 ##

| OS            | IP            | HostName      | Role          |
| ------------- |:-------------:| ------------- | ------------- |
| CentOS6.5_x64 | 192.168.1.232 | node1         | Zookeeper、Codis Server、Codis Dashboard、Codis FE、Sentinel
| CentOS6.5_x64 | 192.168.1.208 | node2         | Zookeeper、Codis Server、Codis Proxy、Sentinel
| CentOS6.5_x64 | 192.168.1.237 | node3         | Zookeeper、Codis Server、Codis Proxy、Sentinel


## 基础环境 ##

<pre>
# 关闭防火墙，所有节点上执行
[root@node1 ~]# service iptables stop 
[root@node1 ~]# chkconfig iptables off
[root@node1 ~]# setenforce 0
[root@node1 ~]# sed -i '/^SELINUX=/{ s/enforcing/disabled/ }' /etc/selinux/config
</pre>

## Zookeeper集群 ##

<pre>
# Codis需要Zookeeper集群来进行协调
# 具体操作步骤，请看Zookeeper集群部署的MarkDown
</pre>

## 安装Codis ##

<p>在需要安装Codis的节点上操作，这里也就是node1、node2、node3。</p>
<pre>
# 基础依赖环境
[root@node1 ~]# yum install -y git gcc make g++ gcc-c++ automake openssl-devel zlib-devel

# 安装GOLANG环境
[root@node1 ~]# wget http://www.golangtc.com/static/go/1.7.1/go1.7.1.linux-amd64.tar.gz
[root@node1 ~]# tar -zxvf go1.7.1.linux-amd64.tar.gz -C /usr/local/
[root@node1 ~]# mkdir -p /home/codis/gopath
[root@node1 ~]# vi /etc/profile   # 在末尾添加以下内容
export GOROOT=/usr/local/go       # GO的安装路径,官方包路径根据这个设置自动匹配
export GOPATH=/home/codis/gopath  # GO的工作路径
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin

[root@node1 ~]# source /etc/profile
[root@node1 ~]# go version   # 查看GOLANG环境是否安装成功
go version go1.7.1 linux/amd64

# 下载并安装Codis
[root@node1 ~]# go get -u -d github.com/CodisLabs/codis  # 由于国内的大环境问题（大家都懂），所以这里会比较慢
[root@node1 ~]# cd /home/codis/gopath/src/github.com/CodisLabs/codis/
[root@node1 codis]# make
[root@node1 codis]# make gotest   # 测试是否安装成功
[root@node1 codis]# ls bin/   # 安装成功后，会出现bin目录，里面包含了codis的所有组件
assets       codis-dashboard  codis-ha     codis-server         redis-benchmark  version
codis-admin  codis-fe         codis-proxy  codis-server-2.8.21  redis-cli
</pre>

## Codis Server ##
<p>在node1、node2、node3节点部署。</p>
<pre>

# 配置Redis（也就是codis server,Codis安装包中自带）,我在3个节点上都安装redis，每个节点上启动一主一从，总共是6台redis实例
# 重新建立Codis目录
[root@node1 codis]# mkdir -p /usr/local/codis
[root@node1 codis]# cp -rf bin/ /usr/local/codis
[root@node1 codis]# mkdir -p /usr/local/codis/{logs,conf,scripts,db,run}
[root@node1 codis]# mkdir -p /usr/local/codis/db/{redis_data_6379,redis_data_6380}
[root@node1 codis]# cp extern/redis-2.8.21/redis.conf /usr/local/codis/conf/redis6379.conf

<font color=red># 主上为了性能可以关闭持久化（aof和rdb），从上可以开启持久化（aof和rdb）</font>
[root@node1 codis]# vim /usr/local/codis/conf/redis6379.conf    <font color=red># 主配置</font>
daemonize yes
pidfile /usr/local/codis/run/redis6379.pid
port 6379
tcp-backlog 511
timeout 300
tcp-keepalive 60
loglevel notice
logfile "/usr/local/codis/logs/redis6379.log"
databases 16
save ""
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump6379.rdb
dir /usr/local/codis/db/redis_data_6379
slave-serve-stale-data yes
slave-read-only yes
maxmemory 10gb
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
requirepass "123456"
appendonly no
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-entries 512
list-max-ziplist-value 64
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes

<font color=red># 从配置，配置基本和主一样，只需要调整一些参数</font>
[root@node1 codis]# cp /usr/local/codis/conf/redis6379.conf /usr/local/codis/conf/redis6380.conf
[root@node1 codis]# vim /usr/local/codis/conf/redis6380.conf  # 修改如下内容
pidfile /usr/local/codis/run/redis6380.pid
port 6380
logfile /usr/local/codis/logs/redis6380.log
dbfilename dump6380.rdb
dir /usr/local/codis/db/redis_data_6380

<font color=red># 启动Codis Server</font>
[root@node1 codis]# cd /usr/local/codis
[root@node1 codis]# ./bin/codis-server ./conf/redis6379.conf
[root@node1 codis]# ./bin/codis-server ./conf/redis6380.conf
</pre>


## Codis Dashboard ##
<p>在node1节点部署</p>

<p><font color=red>仅在需要启动Codis FE的服务器上操作；并且Dashboard没有自带高可用功能，所以存在单点故障，需要借助脚本或者第三方监控工具进行监控。</font></p>
<pre>
[root@node1 ~]# cd /usr/local/codis
[root@node1 codis]# ./bin/codis-dashboard --default-config | tee ./conf/dashboard.conf    # 生成Dashboard配置文件
[root@node1 codis]# vim conf/dashboard.conf
coordinator_name = "zookeeper"
coordinator_addr = "192.168.1.232:2181,192.168.1.208:2181,192.168.1.237:2181"
product_name = "codis-demo"
product_auth = "123456"   # 如何redis设置了密码，需要在这里填上相对应的密码
admin_addr = "0.0.0.0:18080"
sentinel_quorum = 2
# 配置文件参数说明:
#	<font color=red>coordinator_name： </font>外部存储类型，接受 zookeeper/etcd
#	<font color=red>coordinator_addr：</font> 外部存储地址，也就是ZK集群地址
#	<font color=red>product_name：</font> 集群名称，满足正则 \w[\w\.\-]*
#	<font color=red>product_auth：</font> 集群密码，默认为空
#	<font color=red>admin_addr：</font> RESTful API 端口


# 启动Codis Dashboard
[root@node1 codis]# nohup ./bin/codis-dashboard --ncpu=4 --config=./conf/dashboard.conf --log=./logs/dashboard.log --log-level=WARN &> /dev/null &
# 参数说明:
#	<font color=red>--ncpu=N</font> #最大使用 CPU 个数
#	-c  CONF, <font color=red>--config=CONF</font> #指定启动配置文件
#	-l   FILE, <font color=red>--log=FILE</font> #设置 log 输出文件
#	<font color=red>--log-level=LEVEL</font> #设置 log 输出等级：INFO,WARN,DEBUG,ERROR；默认INFO，推荐WARN

<font color=red># 执行成功后查看日志，出现以下内容说明成功</font>
[root@node1 codis]# more ./logs/dashboard.log.2016-11-24
2016/11/24 09:45:47 main.go:77: [WARN] set ncpu = 4
2016/11/24 09:45:47 topom.go:110: [WARN] create new topom:
{
    "token": "194b7427c9003acc28ddad367899c303",
    "start_time": "2016-11-24 09:45:47.60058874 -0500 EST",
    "admin_addr": "192.168.1.232:18080",
    "product_name": "codis_chanzor",
    "pid": 11992,
    "pwd": "/usr/local/codis",
    "sys": "Linux node1 2.6.32-431.el6.x86_64 #1 SMP Fri Nov 22 03:15:09 UTC 2013 x86_64 x86_64 x86_64 GNU/Linux"
}
2016/11/24 09:45:47 main.go:124: [WARN] create topom with config
coordinator_name = "zookeeper"
coordinator_addr = "192.168.1.232:2181,192.168.1.208:2181,192.168.1.237:2181"
admin_addr = "0.0.0.0:18080"
product_name = "codis_chanzor"
product_auth = "123456"
sentinel_quorum = 2
2016/11/24 09:45:47 topom.go:381: [WARN] admin start service on [::]:18080
</pre>

## Codis Proxy ##
在node2节点部署
<pre>
[root@node2 ~]# cd /usr/local/codis 
[root@node2 codis]# ./bin/codis-proxy --default-config | tee ./conf/proxy.conf   # 生成Proxy配置文件
[root@node2 codis]# vim ./conf/proxy.conf
product_name = "codis-demo"
product_auth = "123456"    # 如何redis设置了密码，需要在这里填上相对应的密码
admin_addr = "0.0.0.0:11080"
proto_type = "tcp4"
proxy_addr = "0.0.0.0:19000"
jodis_name = ""
jodis_addr = "192.168.1.232:2181,192.168.1.208:2181,192.168.1.237:2181"
jodis_timeout = "20s"
jodis_compatible = false
proxy_datacenter = ""
proxy_max_clients = 1000
proxy_max_offheap_size = "1024mb"
proxy_heap_placeholder = "256mb"
backend_ping_period = "5s"
backend_recv_bufsize = "128kb"
backend_recv_timeout = "30s"
backend_send_bufsize = "128kb"
backend_send_timeout = "30s"
backend_max_pipeline = 1024
backend_primary_only = false
backend_primary_parallel = 1
backend_replica_parallel = 1
backend_keepalive_period = "75s"
session_recv_bufsize = "128kb"
session_recv_timeout = "30m"
session_send_bufsize = "64kb"
session_send_timeout = "30s"
session_max_pipeline = 512
session_keepalive_period = "75s"
metrics_report_server = ""
metrics_report_period = "1s"
metrics_report_influxdb_server = ""
metrics_report_influxdb_period = "1s"
metrics_report_influxdb_username = ""
metrics_report_influxdb_password = ""
metrics_report_influxdb_database = ""

# 配置文件参数说明:
#	<font color=red>product_name：</font> 集群名称，参考dashboard参数说明
#	<font color=red>product_auth：</font> 集群密码，默认为空
#	<font color=red>admin_addr：</font> RESTfulAPI 端口
#	<font color=red>proto_type：</font> Redis 端口类型，接受tcp/tcp4/tcp6/unix/unixpacket
#	<font color=red>proxy_addr：</font> Redis 端口地址或者路径
#	<font color=red>jodis_addr：</font> Jodis 注册 zookeeper地址
#	<font color=red>jodis_timeout：</font> Jodis 注册 sessiontimeout时间，单位second
#	<font color=red>backend_ping_period：</font> 与codis-server 探活周期，单位second，0表示禁止
#	<font color=red>session_max_timeout：</font> 与 client 连接最大读超时，单位second，0表示禁止
#	<font color=red>session_max_bufsize：</font> 与 client 连接读写缓冲区大小，单位byte
#	<font color=red>session_max_pipeline：</font> 与 client 连接最大的pipeline 大小
#	<font color=red>session_keepalive_period：</font> 与 client 的 tcp keepalive 周期，仅tcp有效，0表示禁止


# 启动Codis Proxy
[root@node2 codis]# nohup ./bin/codis-proxy --ncpu=4 --config=./conf/proxy.conf --log=./logs/proxy.log --log-level=WARN &> /dev/null &
# 参数说明:
# 和Codis Dashboard一样

<font color=red>
# 总结：
# 	如果数据量过大，单台Proxy会成为瓶颈，所以Proxy本身需要负载均衡
# 	对于同一个业务集群而言，可以同时部署多个codis-proxy 实例；
# 	不同 codis-proxy 之间由 codis-dashboard 保证状态同步。
# 	由于Codis支持多台Proxy,所以只需要考虑如何对Proxy进行轮询,这里可以采用LVS+Keepalived、HAproxy、Nginx等常用负载均衡方案；
#	或者云服务商的负载均衡服务，如：阿里的SLB；亦或者在代码层实现。
</font>
</pre>

## Codis FE ##

<p>在node1上部署</p>
<pre>
[root@node1 ~]# cd /usr/local/codis
[root@node1 codis]# ./bin/codis-admin --dashboard-list --zookeeper=192.168.1.232 | tee ./conf/codis.json   # 生成FE配置文件
<font color=red># 注意： --zookeeper  填写ZK集群中任意一台ZK所在的服务器IP地址</font>

[root@node1 codis]# nohup ./bin/codis-fe --ncpu=4 --log=./logs/fe.log --log-level=WARN --dashboard-list=./conf/codis.json --listen=192.168.1.232:18090 &> /dev/null &
# 参数说明:
# --listen=ip:port  填写部署FE服务器的IP和端口
</pre>

## Codis组件连接（FE方式） ##

<p><font color=red>打开浏览器访问http://192.168.1.232:18090,通过FE管理界面操作codis</font></p>



1. 创建Redis Server Group，设置3组
![](http://i.imgur.com/JGeAxOW.png)
2. 添加Redis实例，添加3组，共6台
![](http://i.imgur.com/jnYoQiM.png)
3. 主从同步
![](http://i.imgur.com/ERNeRp1.png)
4. Slot分组
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;把所有Slot分成3组，每组Slot对应一组Redis Server Group。</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Codis中采用预分片的形式，启动的时候就创建了1024个slot，1个slot相当于1个箱子，每个箱子有固定的编号，范围是0~1023。slot这个箱子用作存放Key，至于Key存放到哪个箱子，可以通过算法“crc32(key)%1024”获得一个数字，这个数字的范围一定是0~1023之间，Key就放到这个数字对应的slot。例如，如果某个Key通过算法“crc32(key)%1024”得到的数字是5，就放到编码为5的slot（箱子）。1个slot只能放1个Redis Server Group，不能把1个slot放到多个Redis Server Group中。1个Redis Server Group最少可以存放1个slot，最大可以存放1024个slot。因此，Codis中最多可以指定1024个Redis Server Group。</p>
![](http://i.imgur.com/Ksi7pVw.png)
5. 添加Proxy
![](http://i.imgur.com/jdURNZI.png)

## Codis HA ##

<p>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Codis引入了Redis Server Group，其通过指定一个主Redis和一个或多个从Redis，实现了Redis集群的高可用。当一个主Redis挂掉时，Codis不会自动把一个从Redis提升为主Redis，这涉及数据的一致性问题（Redis本身的数据同步是采用主从异步复制，当数据在主Redis写入成功时，从Redis是否已读入这个数据是没法保证的），需要管理员在管理界面上手动把从Redis提升为主Redis。
</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如果觉得麻烦，Codis也提供了一个工具Codis-HA，这个工具会在检测到主Redis挂掉的时候将其下线，并将一个从Redis提升为主。</p>
<p>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;需要注意，Codis HA将其中一个 slave 升级为 master 时，该组内其他 slave 实例是不会自动改变状态的，这些 slave 仍将试图从旧的 master 上同步数据，因而会导致组内新的 master 和其他 slave 之间的数据不一致。因此当出现主从切换时，需要管理员手动创建新的 sync action 来完成新 master 与 slave 之间的数据同步（codis-ha 不提供自动操作的工具，因为这样太不安全了）。</p>
<pre>
[root@node3 codis]# nohup ./bin/codis-ha  --log=./logs/ha.log  --log-level=WARN  --dashboard=192.168.1.232:18080 &> /dev/null &

# 参数详解
#	 --dashboard： codis dashboard的所在地址
</pre>


## Sentinel ##
<pre>
# 在需要配置哨兵的机器上操作

[root@node2 codis]# vim conf/sentinel.conf
port 26379
bind 0.0.0.0
daemonize yes
logfile "/usr/local/codis/logs/sentinel.log"
pidfile "/usr/local/codis/run/sentinel.pid"


# 参数详解:
<font color=red># requirepass:</font> 设置连接master和slave时的密码，注意的是sentinel不能分别为master和slave设置不同的密码，因此master和slave的密码应该设置相同。
<font color=red># 配置文件里只需要填写基本的参数即可，redis的主从关系不需要填写，因为redis的主从关系是在Codis Dashboard维护的，所以在启动Sentinel时Codis会自动填写主从关系。</font>
[root@node1 codis]# ./bin/codis-server conf/sentinel.conf --sentinel
</pre>

## 正确关闭Codis的姿势 ##

<pre>
# 关闭proxy，如果配置有密码，需要加-a参数在后面接上密码
[root@node2 codis]# ./bin/codis-admin --proxy=192.168.1.208:11080 --shutdown -a 123456
# 原型：
# 	./bin/codis-admin --proxy=ip:port --shutdown

# 关闭dashboard
[root@node1 codis]# ./bin/codis-admin --dashboard=192.168.1.232:18080 --shutdown
# 原型：
# 	./bin/codis-admin --dashboard=ip:port --shutdown
</pre>

## 总结 ##

