---
title: Remote Connect MySQL Server from Client Machine Setup
date: 2020-11-25 15:25:09
categories: [Database, BI]
tags: [Linux, MySQL, Config, Database]
toc: true
cover: /img/mysqlribbon.png
thumbnail: /img/mysqlicon.png
---

You can only connect to MySQL Server from localhost after MySQL installation by default, but in production, all MySQL clients remotely connect to server, for simulating real production environment in your home network, some configurations need to be made to be able to let you connect MySQL from client machine other than localhost.

## Revise or create MySQL configuration file (RHEL or Centos 7)

### Modify or create /etc/my.cnf file

```bash
vim /etc/my.cnf
```

### add a configuration item **bind-address** and let it value to be your MySQL server host ip address (eg. 192.168.1.114)

```bash
bind-address=192.168.1.114
```

<!-- more -->

[![vim_etc_my_cnf.png](/img/screenshots/vim_etc_my_cnf.png)

### save and exit

```bash
:wq
```

## Restart MySQL service

```bash
systemctl restart mysql.service
```

## Open TCP port 3306 using ***iptables***

### Setup `/sbin/iptables` and let firewall opens port on 3306 for any remote machine 

```bash
/sbin/iptables -A INPUT -i eth0 -p tcp --destination-port 3306 -j ACCEPT
```

### To specific client host machine to access port 3306, you can explicitly assign ip address (eg. 192.168.1.134)

```bash
/sbin/iptables -A INPUT -i eth0 -s 192.168.1.134/24 -p tcp --destination-port 3306 -j ACCEPT
```

### Finally save IPv4 firewall rules

```bash
/sbin/iptables-save > /etc/sysconfig/iptables
```

### If it doesn't work, for testing and develop environment, you can turn off firewall

```bash
systemctl stop firewalld.service
systemctl disable firewalld.service
```

## Grant remote access to new MySQL database

### Create a new database

```mysql
create database foo;
```

### Grant remote user access to a specific database on user host machine (192.168.1.134)

```mysql
grant all privileges on foo.* to herman@'192.168.1.134' identified by 'youOwnPasswd';
```

## Grant remote access to existing MySQL database for user (herman) on its host machine (eg. 192.168.1.134)

### Require a set of two commands

```mysql
update db set Host='192.168.1.134' where Db='mysql';
update user set Host='192.168.1.134' where user='herman';
```

## Open MySQL client tool on workstation with address `192.168.1.134` like MySQL workbench

## Create new connection using username and password

### username: herman

### password: youOwnPasswd

### port: 3306

[![mysql_conn_2_remoteSrv.png](/img/screenshots/mysql_conn_2_remoteSrv.png)

## Connection is created and foo database is on the list under user `herman` 

[![mysql_conn_established.png](/img/screenshots/mysql_conn_established.png)




















