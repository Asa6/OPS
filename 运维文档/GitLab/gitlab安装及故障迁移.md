# GitLab #
版本： GitLab 7.12.0   <br />
官方文档地址： https://www.gitlab.com.cn/installation/


## GitLab实战 ##

下列描述均以操作系统 Centos7 为例部署。

步骤：

- 1 环境介绍
- 2 准备工作
- 3 安装配置 Gitlab
- 4 备份
- 5 恢复
- 6 调整配置


### 1 环境介绍 ###

| OS            | IP            | HostName      | Role |
| ------------- |:-------------:| ------------- | -------- |
| CentOS7 x64   | 192.168.10.10   | node1         | GitLab恢复机器 |
| CentOS7 x64   | 10.10.1.18   | localhost         | GitLab机器 |

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

### 3 安装配置 GitLab ###

<pre>
# 添加GitLab 镜像源
[root@node1 ~]# curl -sS http://packages.gitlab.com.cn/install/gitlab-ce/script.rpm.sh | sudo bash

# 查看Gitalb版本
# 选择相应版本并安装 yum install gitlab-ce-&lt;Version&gt;
[root@node1 ~]# yum list gitlab-ce  --showduplicates | sort -r

# 安装Gitalb
[root@node1 ~]# yum -y install gitlab-ce-7.12.0~omnibus-1

# 配置并启动Gitlab
[root@node1 ~]# gitlab-ctl reconfigure
</pre>

打开游览器访问http://192.168.100.10看到gitlab页面即成功
![](http://i.imgur.com/B9MQug4.png)

<br />

### 4 备份 ###
<pre>
# 执行完备份命令后会在/var/opt/gitlab/backups目录下生成备份后的文件
[root@localhost ~]# gitlab-rake gitlab:backup:create
</pre>


### 5 恢复 ###

<pre>
#　将备份文件拷贝到恢复机器的/var/opt/gitlab/backups下。如果backups目录下有多个备份文件，需要指定备份文件，如下所示。(备份和恢复的gitlab版本需保持一致)。
[root@localhost opt]# cd /var/opt/gitlab/backups/
[root@localhost backups]# ls
1502381956_gitlab_backup.tar

[root@node1 ~]# scp root@10.10.1.18:/var/opt/gitlab/backups/1502381956_gitlab_backup.tar /var/opt/gitlab/backups/
[root@node1 ~]# chown git 1502381956_gitlab_backup.tar     # 设置权限

# 停止相关数据连接服务
[root@node1 ~]# gitlab-ctl stop unicorn         
[root@node1 ~]# gitlab-ctl stop sidekiq

[root@node1 ~]# gitlab-rake gitlab:backup:restore BACKUP=1502381956　　#　从备份的1502381956文件恢复
[root@node1 ~]# gitlab-rake gitlab:backup:restore　　　　　　　　　　　　#　backups目录下只有一个备份文件时使用

# 启动gitlab
[root@node1 ~]# gitlab-ctl start
</pre>

### 6 调整配置 ###
<pre>
[root@node1 ~]# cat /etc/gitlab/gitlab.rb |grep -v "^#" | grep -v "^$"   # 修改如下参数的值,请安实际情况填写
external_url 'http://10.10.1.18:8200'
gitlab_rails['gitlab_ssh_host'] = 'xxxxxxxxx'
gitlab_rails['gitlab_email_from'] = 'xxxxxxxxx'
gitlab_rails['gitlab_shell_ssh_port'] = xxxxx
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "xxxxx"
gitlab_rails['smtp_port'] = xxxxxx
gitlab_rails['smtp_user_name'] = "xxxxxxx"
gitlab_rails['smtp_password'] = "xxxxx"
gitlab_rails['smtp_domain'] = "xxxxxx"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true

[root@node1 ~]# gitlab-ctl reconfigure  # 重新配置gitlab
</pre>

打开游览器访问http://http://192.168.100.10:8200,输入用户名和密码后，看到项目http（10.10.1.18:8200）和ssh（prj.testin.cn:8201）地址如下图所示即成功：

![](http://i.imgur.com/IrdaaSt.png)

![](http://i.imgur.com/Hf0mIS6.png)
