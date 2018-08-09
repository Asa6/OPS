# Grafana #
作者：HY.Geng<br />
日期：2017年03月10日<br />
版本：Grafana 3.1.1<br />
官方文档：<a href="http://docs.grafana.org">http://docs.grafana.org</a><br />


## Grafana简介 ##

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Grafana，一个非常好用的开源监控（应该说是出图）软件。各类个性化定制非常易用，像常见的cpu,mem,mysql常用监控项都已经内置。

## Grafana实战 ##

下列描述均以操作系统 Centos7 为例部署,涉及Grafana和Zabbix-Server两台服务器

步骤：

- 1 环境介绍
- 2 准备工作
- 3 安装配置 Grafana
- 4 Grafana + Zabbix 配合使用

### 1 环境介绍 ###

| OS            | IP            | HostName      | Role |
| ------------- |:-------------:| ------------- | -------- |
| CentOS7 x64 | 192.168.0.211| node1         | Zabbix-Server |
| CentOS7 x64 | 192.168.1.192 | node2         | Grafana-Server |

### 2 准备工作 ###
<pre>
# 关闭Iptables和SELinux
[root@node2 ~]# systemctl stop firewalld
[root@node2 ~]# systemctl disable firewalld
[root@node2 ~]# setenforce 0
[root@node2 ~]# sed -i '/^SELINUX=/{ s/enforcing/disabled/ }' /etc/selinux/config

# 调整服务器时间
[root@node2 ~]# yum -y install ntp
[root@node2 ~]# ntpdate -u 202.120.2.101
</pre>

<br />

### 3 安装配置 Grafana ###
<pre>
[root@node2 ~]# vim /etc/yum.repos.d/grafana.repo   添加yum源
[grafana]
name=grafana
baseurl=https://packagecloud.io/grafana/stable/el/6/$basearch
repo_gpgcheck=1
enabled=1
gpgcheck=0
gpgkey=https://packagecloud.io/gpg.key https://grafanarel.s3.amazonaws.com/RPM-GPG-KEY-grafana
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt

[root@node2 ~]# yum list --showduplicates| grep ^grafana   # 查看grafana的各个版本，最新grafana版本已到4.x.x
grafana.x86_64                            2.0.1-1                      grafana  
grafana.x86_64                            2.0.2-1                      grafana  
grafana.x86_64                            2.1.0-1                      grafana  
grafana.x86_64                            2.1.1-1                      grafana  
grafana.x86_64                            2.1.2-1                      grafana  
grafana.x86_64                            2.1.3-1                      grafana  
grafana.x86_64                            2.5.0-1                      grafana  
grafana.x86_64                            2.6.0-1                      grafana  
grafana.x86_64                            3.0.1-1                      grafana  
grafana.x86_64                            3.0.2-1463383025             grafana  
grafana.x86_64                            3.0.3-1463994644             grafana  
grafana.x86_64                            3.0.4-1464167696             grafana  
grafana.x86_64                            3.1.0-1468321182             grafana  
grafana.x86_64                            3.1.1-1470047149             grafana  
grafana.x86_64                            4.0.0-1480439068             grafana  
grafana.x86_64                            4.0.1-1480694114             grafana  
grafana.x86_64                            4.0.2-1481203731             grafana  
grafana.x86_64                            4.1.0-1484127817             grafana  
grafana.x86_64                            4.1.1-1484211277             grafana  
grafana.x86_64                            4.1.2-1486989747             grafana

# 由于zabbix和grafana配合使用时，需要用到Zabbix plugin for Grafana这个插件，而目前这个插件需要依赖Grafana3.x.x版本，所以我们安装Grafana3.x.x版本，而不是最新的Grafana4.x.x版本
[root@node2 ~]# yum -y install grafana-3.1.1-1470047149

[root@node2 ~]# systemctl daemon-reload    # 重载配置文件
[root@node2 ~]# systemctl start grafana-server  # 启动grafana-Server
[root@node2 ~]# systemctl status grafana-server  # 查看grafana-Server状态
[root@node2 ~]# systemctl enable grafana-server.service   # 设置开机启动
</pre>

完成上面步骤后，打开游览器访问http://192.168.1.192:3000，如下图：
![](http://i.imgur.com/yNha3ge.png)

输入默认用户名：admin 密码：admin 后登陆，如下图：
![](http://i.imgur.com/EEbnAzn.png)

<br />

### 4 Grafana + Zabbix 配合使用 ###

<pre>
# 在Grafana-Server中安装Zabbix plugin for Grafana插件
[root@node2 ~]# grafana-cli plugins install alexanderzobnin-zabbix-app
installing alexanderzobnin-zabbix-app @ 3.3.0
from url: https://grafana.net/api/plugins/alexanderzobnin-zabbix-app/versions/3.3.0/download
into: /var/lib/grafana/plugins

✔ Installed alexanderzobnin-zabbix-app successfully 

Restart grafana after installing plugins . <service grafana-server restart>
# 出现successfully等字眼，说明成功

# 重启Grafana-Server
[root@node2 ~]# systemctl restart grafana-server
</pre>

重新访问http://192.168.1.192:3000，点击左上角的logo，选择下拉框中的Plugins，如下图：
![](http://i.imgur.com/aKW2xJh.png)

![](http://i.imgur.com/MbbZcyt.png)

选择Apps按钮，并点击Zabbix这个APP后，如下图：
![](http://i.imgur.com/f6XWU4d.png)

![](http://i.imgur.com/RFi2rKg.png)

点击Enable后，如下图：
![](http://i.imgur.com/fzPdIxm.png)

然后点击，左上角logo，选择下拉框中的Data Sources后，如下图：
![](http://i.imgur.com/790drRF.png)

选择Add data source,填写相关参数并Add，如下图：
![](http://i.imgur.com/YmXeE0q.png)
<font color=#FF0000>这里点击Add后，一定要看到Sucess，否则Grafana不能正确获取Zabbix数据！！！</font>

![](http://i.imgur.com/entQi8G.png)
参数详解具体请看官方解释：<a href="http://docs.grafana-zabbix.org/installation/configuration/">http://docs.grafana-zabbix.org/installation/configuration/</a><br />

然后在Zabbix中，添加一台被监控的主机，名为client1，如图：
![](http://i.imgur.com/5DbWU1m.png)

![](http://i.imgur.com/QQ0GvZf.png)

点击左上角的logo，选择Dashboards中的home，并选择默认模板Template Linux Server，选择正确的Group、Host。如果一切顺利，即可看到监控图形：
![](http://i.imgur.com/NF64wTD.png)

可以看到，Grafana成功的把Zabbix采集的数据，用图形显示了！！！

<br />

## 番外篇——更改Grafana主题 ##
点击左上角logo，选择admin中的Profile，如下图：
![](http://i.imgur.com/6Gz32db.png)

![](http://i.imgur.com/OoKf6O1.png)

选择Preferences中的UT Theme中的Light，并点击Update，如图：
![](http://i.imgur.com/vJEue7V.png)

这样主题就更改成功了，看下面的效果图：
![](http://i.imgur.com/jPIhn72.png)

![](http://i.imgur.com/fndxKqA.png)


到此，全部结束！
