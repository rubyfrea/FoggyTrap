---
layout: post
title:  "MySQL Replication 入门"
date:   2016-05-24 14:54:58 +0800
categories: 数据库
---

本系列主题是数据库高可用方案，从MySQL出发，遍历rdbms，再过渡到非关系型数据库。
关系型数据库的研究范围：
![主题]({{site.url}}/image/db1theme.png)
Line
MySQL Replication是MySQL保持数据冗余的一种方式，常用的结构有Master-Slave，Multi-Master，Multi-Source几种方式。

如果Master发生故障，可以立刻将一台Slave切换成Master，Replicaiton则是这个过程的基础，但故障转移还是要结合一个管理层来完成，比如ZooKeeper、Keepalive。

我们先从最简单的master-slave开说。OS用的Ubuntu 14.04。


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

#安装mysql最新版本
#download deb file at http://dev.mysql.com/downloads/repo/apt/
    wget http://dev.mysql.com/get/mysql-apt-config_0.7.2-1_all.deb  
    sudo dpkg -i mysql-apt-config_0.7.2-1_all.deb  
    sudo apt-get update
    sudo apt-get install mysql-server

#配置简单的主从

#在server 1端
    sudo su
    vi /etc/mysql/my.cnf
    
#修改 my.cnf
    [mysqld]
    log-bin = mysql-bin
    server-id = 1
    innodb_flush_log_at_trx_commit = 1
    sync_binlog = 1
    binlog_do_db = logdb 
#comment the following line
    bind-address = 127.0.0.1
#保存 my.cnf
#重启
    service mysql restart
#进入mysql shell
    mysql -u root -proot  
#创建远程连接用户
    mysql>  create user 'replic'@'%' identified by 'replicpassword';
    mysql>  GRANT REPLICATION SLAVE ON *.* TO 'replic'@'%' IDENTIFIED BY 'replicpassword';   
#先锁住log
    mysql>  FLUSH TABLES WITH READ LOCK;   
#记录结果binlon
    mysql>  SHOW MASTER STATUS;    
#解锁
    mysql> unlock tables;    
#在server2上
#修改my.cnf
    sudo su
    vi /etc/mysql/my.cnf
#修改以下行
    server-id   = 2
    log-bin = mysql-bin
    innodb_flush_log_at_trx_commit = 1
    sync_binlog = 1
    binlog_do_db = logdb
#注释掉下一句
    bind-address= 127.0.0.1
#保存
#重启
    service mysql restart
#进入mysql shell
    mysql -u root -proot
#根据server1 status里的信息在server2端配置master
    mysql> stop slave;
    mysql> change master to master_host = '172.16.13.211', master_user = 'replic', master_password = 'replicpassword', master_log_file = '之前在server1中记录的log file', master_log_pos = 在server1记录的log编号；
    mysql> start slave;

**完成**

可以在
    
