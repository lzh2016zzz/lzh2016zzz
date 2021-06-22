---
title: centos防火墙firewalld常用命令速查
tags: 
   - linux
   - 速查
---


记录一下Centos&RedHat中对firewalld的常用操作命令.以备不时之需.

虽然现在的弹性vps都自带出入站规则.但是偶尔还是会有点用的.

```bash

# 1.查看防火墙状态：

firewall-cmd --state 

# 2.启动防火墙

systemctl start firewalld

# 3.关闭防火墙

systemctl stop firewalld

# 4.检查防火墙开放的端口

firewall-cmd --permanent --zone=public --list-ports

# 5.开放一个新的端口

firewall-cmd --zone=public --add-port=80/tcp --permanent

# 6.重启防火墙

firewall-cmd --reload

# 7.验证新增加端口是否生效

firewall-cmd --zone=public --query-port=443/tcp

# 8.防火墙开机自启动

systemctl enable firewalld.service

# 9.防火墙取消某一开放端口

firewall-cmd --zone=public --remove-port=443/tcp --permanent

```
