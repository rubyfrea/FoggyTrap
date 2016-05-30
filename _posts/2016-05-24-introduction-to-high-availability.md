---
layout: post
title:  "数据库高可用方案分析"
date:   2016-05-24 14:54:58 +0800
categories: jekyll 数据库
---
如果对以下主题不感兴趣，可以移步Google啦。不占用大家伙时间。

![主题]({{site.url}}/image/db1theme.png)
Line
MySQL Replication是MySQL保持数据冗余的一种方式，常用的结构有Master-Slave，Multi-Master，Multi-Source几种方式。

如果Master发生故障，可以立刻将一台Slave切换成Master，Replicaiton则是这个过程的基础，但故障转移还是要结合一个管理层来完成，比如ZooKeeper、Keepalive


{% highlight Lang lineanchors %}


#安装环境


# 安装 java,maven
sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java7-installer

# 设置默认 jdk
sudo update-alternatives --config java

# 配置 Java Home 编辑 ~/.bashrc
JAVA_HOME=/usr/lib/jvm/java-7-oracle
export JAVA_HOME
PATH=$PATH:$JAVA_HOME
export PATH

# maven
sudo apt-get remove maven2
sudo apt-get update
sudo apt-get install maven # 安装 maven3，2 在之后会有 bug

#安装mysql最新版本
#download deb file at http://dev.mysql.com/downloads/repo/apt/
wget http://dev.mysql.com/get/mysql-apt-config_0.7.2-1_all.deb  
sudo dpkg -i mysql-apt-config_0.7.2-1_all.deb  
sudo apt-get update
sudo apt-get install mysql-server

#按照以上配置好两台机器

#在server 1上

#修改my.cnf
sudo su
vi /etc/mysql/my.cnf
#修改以下行

#bind-address= 127.0.0.1
server-id               = 1
log_bin                 = /var/log/mysql/mysql-bin.log
binlog_do_db            = testdb_512

#保存

#进入mysql shell
mysql -u root -proot

#先锁住log
mysql>   FLUSH PRIVILEGES;
mysql>  USE testdb_512;
mysql>  FLUSH TABLES WITH READ LOCK;

#创建远程连接用户
mysql>  create user 'replic'@'%' identified by 'replicpassword';
mysql>  GRANT REPLICATION SLAVE ON *.* TO 'replic'@'%' IDENTIFIED BY 'replicpassword';

mysql> unlock tables;

#记录结果
mysql>  SHOW MASTER STATUS;

#在server2上

#修改my.cnf
sudo su
vi /etc/mysql/my.cnf
#修改以下行

#bind-address= 127.0.0.1
server-id               = 2
log_bin                 = /var/log/mysql/mysql-bin.log
binlog_do_db            = testdb_512

#保存

#进入mysql shell
mysql -u root -proot

#创建远程连接用户
mysql> mysql -h master <
mysql>  create user 'replic'@'%' identified by 'replicpassword';
mysql>  GRANT REPLICATION SLAVE ON *.* TO 'replic'@'%' IDENTIFIED BY 'replicpassword';
mysql>   FLUSH PRIVILEGES;
mysql>  USE testdb_512;
mysql>  FLUSH TABLES WITH READ LOCK;
#记录结果binlon
mysql>  SHOW MASTER STATUS;
mysql> stop slave;
mysql> change master to master_host = '172.16.13.214', master_user = 'replic', master_password = 'replicpassword', master_log_file = '之前在server1中记录的log file', master_log_pos = 在server1记录的log编号；
mysql> start slave;
mysql> unlock tables;

#在server1端
mysql> stop slave;
mysql> change master to master_host = '172.16.13.214', master_user = 'replic', master_password = 'replicpassword', master_log_file = '之前在server2中记录的log file', master_log_pos = 在server2记录的log编号；
mysql> start slave;

#测试互为主从是否运行成功
#在server1端建个数据库
mysql> create database example1 
mysql> use example1;
mysql> create table student(`id` varchar(20));

#在server2端查看，同理在server2端新建，在server1端查看




#可能会用到的命令

#view iptables
sudo iptables -L

#配置网卡
sudo su
ifconfig eth0

#configure dns
sudo vi /etc/network/interfaces

#revise the line 'dns-nameservers X.X.X.X'
#when done
sudo ifdown eth0 && sudo ifup eth0if
#或者
sudo ifdown --exclude=lo -a && sudo ifup --exclude=lo -a
find / -name dns
# /etc/resolv.conf which resolvconf


#test percona xtraDB CLUSTER
#INSTALL XTRADB
apt-key adv --keyserver keys.gnupg.net --recv-keys 1C4CBDCDCD2EFD2A
#add following shell to /etc/apt/sources.list,replacing version with the name of your distribution
deb http://repo.percona.com/apt VERSION main
deb-src http://repo.percona.com/apt VERSION main


#vmware
# change the logical name of the network interface to adapt to one's needs
vi /etc/udev/rules.d/70-persistent-net.rules.
# view ethernet network interface
ifconfig a | grep eth
# if you want to see details, another way of above
sudo lshw -class network

{% endhighlight %}


Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
