---
layout: post
title: "Cloudera CDH Install Guide"
description: "Install Guide of Cloudera CDH"
category: hadoop
tags: [hadoop,cloudera,cdh]
---
{% include JB/setup %}

#Cloudera 安装

## 机器准备

三台服务器

* 1 台主服务器(mccrea)，16G 内存 + 200G 硬盘
* 2 台工作节点(walle + eve)， 8G 内存 + 100G 硬盘

### 硬盘 LVM

	# reference: http://www.cnblogs.com/gaojun/archive/2012/08/22/2650229.html
	
	$ fdisk /dev/vdb # 硬盘分区 (分区格式指定为LVM分区 8e)
	
	$ pvcreate /dev/vdb1 # 创建物理卷
	$ vgcreate vg-hadoop /dev/vdb1 # 创建卷组
	$ lvcreate -l <PE Size> -n lv-hadoop vg-hadoop # 创建逻辑卷
	$ mkfs -t ext3 /dev/vg-hadoop/lv-hadoop # 格式化
	$ echo "/dev/vg-hadoop/lv-hadoop /opt ext3 defaults 1 2" >> /etc/fstab # 添加到 fstab
	$ mount /dev/vg-hadoop/lv-hadoop /opt # 挂载
	
	## 扩容空间
	# 先创建好新的物理卷，如: /dev/vdb2
	$ vgextend vg-hadoop /dev/vdb2 # 将物理卷扩展到卷组
	$ lvextend -l <PE Size> /dev/vg-hadoop/lv-hadoop # 追加指定大小的空间到逻辑卷
	$ resize2fs /dev/vg-hadoop/lv-hadoop # 执行扩容
	$ df -h # 查看扩容结果

### 主机命名

* Reference: http://www.cnblogs.com/Anker/p/3355070.html
* 在 /etc/hosts 中增加 IP 与主机名直接的映射，如：

		10.232.XX.XX walle
		10.221.XX.XX eve
		10.143.XX.XX mccrea
* 修改 /etc/sysconfig/network 中的 HOSTNAME

		NETWORKING=YES
		HOSTNAME=mccrea
* 使用 hostname 命令修改

		$ hostname mccrea
		$ hostname # 查看结果，重新登录
* 修改命令行提示符

		PS1='\u@\H:\w\$ '

### SSH 互联

	$ ssh-key-gen # 生成密钥
	$ ssh-copy-id user@walle # 让远程主机允许使用密钥访问
	$ scp ~/.ssh/id_rsa* user@walle:~/.ssh # 远程复制公钥和私钥
	$ cat ~/.ssh/kd_rsa.pub >> ~/.ssh/authorized_keys # 允许远程主机使用该密钥连接本机


### 防火墙设置

由于集群设计的主机和端口较多，相关软件的安全机制需要注意搭建，所以首先以通过 iptables 进行端口安全控制为主，对需要外部访问的机器辅以授权验证机制。

涉及的主机

| 内网 IP | 名称 |外网 IP | 角色 |
|--------|------|-------|------|
| 10.143.XX.XX | mccrea | 203.195.XX.XX | cloudera server |
| 10.232.XX.XX | walle | 203.195.XX.XX | cloudera agent |
| 10.221.XX.XX | eve | 203.195.XX.XX | cloudera agent |


内部互联互通规则，只对外开放必要的主机和端口，并用密码保护

References:

http://wiki.centos.org/zh/HowTos/Network/IPTables

创建安全脚本 setup-firewall.sh 如下

	#!/bin/bash
	# http://wiki.centos.org/zh/HowTos/Network/IPTables
	# ssh root@MachineB 'bash -s' < local_script.sh
	# use puppet later
		
	declare -A hosts
	hosts=(
	  [walle]=10.232.XX.XX 
	  [eve]=10.221.XX.XX
	  [mccrea]=10.143.XX.XX
	)
	
	declare -A mports
	mports=(
	  [cm_http]=7180
	  [cm_https]=7183 
	  [hue]=8888
	)
	
	#
	# iptables 样例设置脚本
	#
	# 清除 iptables 内一切现存的规则
	#
	 iptables -F
	#
	# 容让 SSH 连接到 tcp 端口 22
	# 当通过 SSH 远程连接到服务器，你必须这样做才能群免被封锁于系统外
	#
	 iptables -A INPUT -p tcp --dport 22 -j ACCEPT
	#
	# 设置 INPUT、FORWARD、及 OUTPUT 链的缺省政策
	#
	 iptables -P INPUT DROP
	 iptables -P FORWARD DROP
	 iptables -P OUTPUT ACCEPT
	#
	# 设置 localhost 的访问权
	#
	 iptables -A INPUT -i lo -j ACCEPT
	#
	# 接纳属于现存及相关连接的封包
	#
	 iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
	#
	# 接纳来自被信任 IP 地址的封包
	if [ "$(hostname)" == "mccrea" ]; then
	  for p in "${!mports[@]}"
	  do
       iptables -A INPUT -p tcp --dport ${mports[$p]} -j ACCEPT
      done
	fi
	
	for h in "${!hosts[@]}"
	do
  	  if [ "$(hostname)" != "$h" ]; then
	    iptables -A INPUT -s ${hosts[$h]} -j ACCEPT
	  fi
	done
	
	# 打开 PING
	iptables -A INPUT -p icmp --icmp-type 8 -i eth0 -j ACCEPT

	# 存储设置
	#
	 /sbin/service iptables save
	#
	# 列出规则
	#
	 iptables -L -v
	 
### 安全密码

使用 Bash 脚本定义如下函数 genpasswd，产生安全密码用于 scm 和 hue 等对外服务

	# reference: http://www.cyberciti.biz/faq/linux-random-password-generator/
	genpasswd() {
	local l=$1
       	[ "$l" == "" ] && l=16
      	tr -dc A-Za-z0-9_ < /dev/urandom | head -c ${l} | xargs
	}
	
	$ genpasswd # 生成 16 位密码
	$ genpasswd 8 # 生成 8 位密码
	
## 安装集群

相关文档下载：

	wget http://www.cloudera.com/content/cloudera/en/documentation/core/latest/PDF/cloudera-introduction.pdf
	wget http://www.cloudera.com/content/cloudera/en/documentation/core/latest/PDF/cloudera-releases.pdf
	wget http://www.cloudera.com/content/cloudera/en/documentation/core/latest/PDF/cloudera-installation.pdf
	wget http://www.cloudera.com/content/cloudera/en/documentation/core/latest/PDF/cloudera-quickstart.pdf
	wget http://www.cloudera.com/content/cloudera/en/documentation/core/latest/PDF/cloudera-administration.pdf
	wget http://www.cloudera.com/content/cloudera/en/documentation/core/latest/PDF/cloudera-datamgmt.pdf
	wget http://www.cloudera.com/content/cloudera/en/documentation/core/latest/PDF/cloudera-operation.pdf
	wget http://www.cloudera.com/content/cloudera/en/documentation/core/latest/PDF/cloudera-security.pdf
	wget http://www.cloudera.com/content/cloudera/en/documentation/core/latest/PDF/cloudera-impala.pdf
	wget http://www.cloudera.com/content/cloudera/en/documentation/core/latest/PDF/cloudera-search.pdf

### Cloudera Manager 

* Save the appropriate Cloudera Manager repo file (cloudera-manager.repo) for your system:

|OS Version|Repo URL|
|----------|--------||Red Hat/CentOS/Oracle 5 | http://archive.cloudera.com/cm5/redhat/5/x86_64/cm/cloudera-manager.repo||Red Hat/CentOS 6|http://archive.cloudera.com/cm5/redhat/6/x86_64/cm/cloudera-manager.repo|
* Copy the repo file to the /etc/yum.repos.d/ directory.
* Install the Oracle JDK, which is included in the Cloudera Manager 5 repositories

		$ sudo yum install oracle-j2sdk1.7 
* Install the Cloudera Manager Server Packages

		$ sudo yum install cloudera-manager-daemons cloudera-manager-server
* Installing the MySQL JDBC Connector
	2. Download the MySQL JDBC connector from http://www.mysql.com/downloads/connector/j/5.1.html.	3. Extract the JDBC driver JAR file from the downloaded file. For example:	4. Add the JDBC driver, renamed, to the relevant server. For example:
			tar zxvf mysql-connector-java-5.1.31.tar.gz		If the target directory does not yet exist on this host, you can create it before copying the JAR file. For example:		
			$ sudo cp mysql-connector-java-5.1.31/mysql-connector-java-5.1.31-bin.jar /usr/share/java/mysql-connector-java.jar			$ sudo mkdir -p /usr/share/java/			$ sudo cp mysql-connector-java-5.1.31/mysql-connector-java-5.1.31-bin.jar /usr/share/java/mysql-connector-java.jar
* Setting up the Cloudera Manager Server Database
CreatedatabasesfortheActivityMonitor,ReportsManager,HiveMetastore,SentryServer,ClouderaNavigator Audit Server, and Cloudera Navigator Metadata Server:
	
		mysql> create database database DEFAULT CHARACTER SET utf8; Query OK, 1 row affected (0.00 sec)		mysql> grant all on database.* TO 'user'@'%' IDENTIFIED BY 'password'; Query OK, 0 rows affected (0.00 sec)


	| Role | Database | User | Password |
|---------|----------|-----------|----------|| Activity Monitor | amon | amon | amon_password || Reports Manager | rman | rman | rman_password || Hive Metastore Server | metastore | hive | hive_password || Sentry Server | sentry | sentry | sentry_password || Cloudera Navigator Audit Server | nav | nav | nav_password || Cloudera Navigator Metadata Server | navms | navms | navms_password |

### 安装准备



