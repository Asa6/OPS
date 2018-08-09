# Elastic Stack(ELK 5) #
作者：耿浩源<br />
日期：2017年2月23日<br />
版本：Elastic Stack5.2<br />
官方文档：<a href="https://www.elastic.co/guide/index.html">https://www.elastic.co/guide/index.html</a><br />
![](http://i.imgur.com/BeGhA8r.png)

<br />

## Elastic Stack简介 ##

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;ELK（Elasticsearch + Logstash + Kibana）用于日志集中分析系统，Elasticsearch 用于存储、搜索、分析数据，Logstash 用于接收并处理数据，Kibana 提供 Web UI 管理数据，客户端通过 Logstash-Forwarder 将指定的日志数据传递数据给 ELK 系统，大体流程如下图：
![](http://i.imgur.com/rWzj2ID.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;后来Elastic 团队收购了 Packetbeat 团队，就建立了 Beat，Beat 是一个轻量级的数据收集平台，可以将不同的数据发送 ELK 系统，例如日志、网络数据、系统信息等等。Elastic 团队在命名时最终将 ELK + Beat 命名为 Elastic Stack，并将整个产品线的版本提升至 5.0。之前使用 ELK 时，几个产品的版本需要对应，例如使用 Elasticsearch 1.6，Logstash 1.5，Kibana 4.1，如果版本没有正确对应，将可能导致无法正常运行。<br />目前 Elastic 团队已将整体产品线都提升到 5.2，这样在部署系统时免去了操心版本对应的事情。

<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Elastic Stack （之后简称 ES）已经不单单用于分析日志，Beat 可以代理更多类型的数据输出，Beat 可以直接输出数据至 Elasticsearch，也可输出数据给 Logstash，再由 Logstash 处理后输出到 Elasticsearch，如下图：

![](http://i.imgur.com/GwxPNOv.png)

![](http://i.imgur.com/Hw2WKfT.png)

<br />
目前发布的 Beats 产品（Beats 的几个产品均为独立发布）：<br />

- Filebeat：收集日志数据
- Packetbeat：收集网络数据
- Metricbeat：收集系统及服务数据（替代Topbeat）
- Winlogbeat：收集 Windows 事件
- Heartbeat: 测监视服务的可用性（目前在测试版，生产慎用）

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Elastic Stack 中还包含一个以独立产品发布的插件 X-Pack，集成了监控、报警、报表及图表的功能。X-Pack 相当于一个插件集合包，简化了以往安装相关功能插件的过程。同时 X-Pack 还提供了管理（包括用户和角色的管理）及监控的 UI 界面。本文的安装配置先不涉及这个插件，以后再独立的文章中来介绍。

<br />
## Elastic Stack实战 ##

下列描述均以操作系统 Centos7 为例部署，涉及服务端和客户端两台服务器，请确保服务端所在服务器内存至少大于 2G。

步骤：

- 1 环境介绍
- 2 准备工作
- 3 安装配置 Elasticsearch
- 4 安装配置 Kibana
- 6 安装配置 Filebeat
- 7 安装配置 Logstash （可选）
- 8 安装配置 Nginx （可选）
- 9 单独配置一个客户端

<br />

### 1 环境介绍 ###

| OS            | IP            | HostName      |Machine configuration  | Role |
| ------------- |:-------------:| ------------- | ------|-------- |
| CentOS7 x64 | 192.168.0.211 | node1         |CPU：2核 Mem：4G | kibana、elasticsearch、logstash、filebeat  |

<br />

### 2.准备工作 ###

<pre>
# 由于测试环境，所以关闭Iptables和SELinux
[root@node1 ~]# systemctl stop firewalld
[root@node1 ~]# systemctl disable firewalld
[root@node1 ~]# setenforce 0
[root@node1 ~]# sed -i '/^SELINUX=/{ s/enforcing/disabled/ }' /etc/selinux/config

# 调整服务器时间
[root@node1 ~]# yum -y install ntp
[root@node1 ~]# ntpdate -u 202.120.2.101

# 安装java环境
[root@node1 ~]# yum -y install java-1.8.0-openjdk.x86_64
[root@node1 ~]# java -version
openjdk version "1.8.0_121"
OpenJDK Runtime Environment (build 1.8.0_121-b13)
OpenJDK 64-Bit Server VM (build 25.121-b13, mixed mode)

# 修改文件句柄限制（默认1024）
[root@node1 ~]# echo "*    soft nofile 65536" >> /etc/security/limits.conf
[root@node1 ~]# echo "*    hard nofile 65536" >> /etc/security/limits.conf
[root@node1 ~]# ulimit -n  # <font color=red>重新打开终端后</font>，使用ulimit -n 查看文件句柄限制
65535
</pre>

<br />

### 3 安装配置 Elasticsearch ###
<pre>
# 导入签名
[root@node1 ~]# rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

# 配置yum源
[root@node1 ~]# vim /etc/yum.repos.d/elasticsearch.repo
[elasticsearch-5.x]
name=Elasticsearch repository for 5.x packages
baseurl=https://artifacts.elastic.co/packages/5.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md

[root@node1 ~]# yum -y install elasticsearch

# Elasticsearch 配置文件在 /etc/elasticsearch/elasticsearch.yml，如果不使用 Logstash 或者 Logstash 与 Elasticsearch 不在同一服务器，<br />那么需要使 Elasticsearch 监听到指定的 IP 地址和端口，修改 elasticsearch.yml 中的下边两行：
[root@node1 ~]# vim /etc/elasticsearch/elasticsearch.yml
network.host: 0.0.0.0  # 监听所有IP
http.port: 9200

[root@node1 ~]# systemctl enable elasticsearch.service
[root@node1 ~]# systemctl start elasticsearch.service
[root@node1 ~]# curl -XGET 'localhost:9200/?pretty'   # 得到类似如下结果，则说明安装正确
{
  "name" : "5191ncM",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "840feU0mQSaN-PDYjhDb8g",
  "version" : {
    "number" : "5.2.1",
    "build_hash" : "db0d481",
    "build_date" : "2017-02-09T22:05:32.386Z",
    "build_snapshot" : false,
    "lucene_version" : "6.4.1"
  },
  "tagline" : "You Know, for Search"
}
</pre>

<br />

### 4 安装配置 Kibana ###
<pre>
[root@node1 ~]# vim /etc/yum.repos.d/kibana.repo
[kibana-5.x]
name=Kibana repository for 5.x packages
baseurl=https://artifacts.elastic.co/packages/5.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md

[root@node1 ~]# yum -y install kibana

# 编辑配置文件 /etc/kibana/kibana.yml，修改如下两行:
[root@node1 ~]# vim /etc/kibana/kibana.yml
server.port 5601
server.host 0.0.0.0

[root@node1 ~]# systemctl enable kibana.service
[root@node1 ~]# systemctl start kibana.service
</pre>

浏览器打开 http://192.168.0.211:5601，看到如下界面说明 Kibana 安装正确。

![](http://i.imgur.com/ZnqtorL.png)

<br />

### 5 安装配置 Filebeat ###

<pre>
# 已经添加过es的YUM源了，直接yum安装
[root@node1 ~]# yum -y install filebeat

# Filebeat 默认配置会传输 /var/log/*.log （对于 Centos 系统来说，这下边至少会有 yum.log）传输至本地的 Elasticsearch，<br />因此不需要修改任何配置，目前 ES 系统已经可以收集日志数据了。
</pre>
在浏览器中打开 http://192.168.0.211:5601,将 index name or pattern 设置为 **filebeat-***，点击 create 按钮，如下图：
![](http://i.imgur.com/jkj0HJy.png)

然后点击Discover按钮，如果一切顺利，就可以看到 ES 系统已经收集到日志数据了，如下图：
![](http://i.imgur.com/F8T4ZYy.png)

<br />

### 6 安装配置 Logstash ###
<pre>
# 已经添加过es的YUM源了，直接yum安装
[root@node1 ~]# yum -y install logstash

# 配置 Logstash 的输入和输出，新建输入输出配置文件：
[root@node1 ~]# vi /etc/logstash/conf.d/first-logstash.conf
input {
  beats {
    port => 5044
  }
}

output {
  elasticsearch {
    hosts => "localhost:9200"   # 填写elasticsearch的地址，这里由于Logstash和elasticsearch在一台机器上，所以默认即可
  }
}
# 输入（input）：设置监听 5044 端口，接收 beats 的输入数据
# 输出(output)：将数据输出到 Elasticsearch
</pre>

<pre>
# 启动logstash
[root@node1 ~]# systemctl start logstash
</pre>

#### 6.1 修改 Filebeat 将日志发送给 Logstash ####

filebeat 可以将日志输入到 Elasticsearh，如刚才的配置。它也可以将日志输入给 Logstash，由 Logstash 处理日志，再将处理后的日志数据输入到 Elasticsearch。下边配置 Filebeat 将日志 输入到 Logstash。
<pre>
# 注释掉 Elasticsearch output 的相关设置：
[root@node1 ~]# vi /etc/filebeat/filebeat.yml
#-------------------------- Elasticsearch output ------------------------------
#output.elasticsearch:
  # Array of hosts to connect to.
#  hosts: ["localhost:9200"]

  # Optional protocol and basic auth credentials.
  #protocol: "https"
  #username: "elastic"
  #password: "changeme"
</pre>

<pre>
# 配置输出到 Logstash：
[root@node1 ~]# vi /etc/filebeat/filebeat.yml
#----------------------------- Logstash output --------------------------------
output.logstash:
  # The Logstash hosts
  hosts: ["localhost:5044"]

  # Optional SSL. By default is off.
  # List of root certificates for HTTPS server verifications
  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

  # Certificate for SSL client authentication
  #ssl.certificate: "/etc/pki/client/cert.pem"

  # Client Certificate Key
  #ssl.key: "/etc/pki/client/cert.key"
</pre>

<pre>
# 重启filebeat
[root@node1 ~]# systemctl restart filebeat
</pre>

<br />

## 关于日志显示问题 ##

有时日志文件里有如下图这样的日志：
![](http://i.imgur.com/QqrabJz.png)

而在kibana页面中看到是的是如下图这样的：
![](http://i.imgur.com/q9nktdJ.png)

ELK把这一段错误日志分成一行一行显示了，这并不是我们想要，我们需要把这段错误日志在一块显示。

解决办法：
<pre>
# 打开logstash的输入输出配置文件，在beats段中加入这样的配置:
codec => multiline {
            pattern => "^\["
            negate => true
            what => "previous"
        }
</pre>

<pre>
# 修改后的输入输出配置文件如下：
[root@node1 ~]# cat /etc/logstash/conf.d/first-logstash.conf
input {
  beats {
    port => 5044
    codec => multiline {
            pattern => "^\["
            negate => true
            what => "previous"
        }
  }
}

output {
  elasticsearch {
    hosts => "localhost:9200"
  }
}
# 注意：解析时，最后一行日志，不会解析。只有当再追加一条日志时，才会解析最后一条日志。（也就是说每次产生新的日志时，才会把上一次的最后一行在kibana上显示）
</pre>


再次在日志文件中打印刚才的报错日志，在kibana页面中看到如下图：
![](http://i.imgur.com/CZQcxO3.png)

可以看到，已经变成一行！