---
title: 在linux下配置swap分区
tags: 经验,读书笔记,linux
---


#### 如何在linux下配置swap分区

简单配置,做个备忘方便以后自己查阅��用.

##### 1.创建swap文件

```bash
#在/var/swap下创建swap文件
#大小是2048000bytes
#/var/swap 可以替换成自己喜欢的地址
$ dd if=/dev/zero of=/var/swap bs=1024 count=2048000
```


##### 2.设置swap文件

```bash
#格式化swap文件
$ mkswap -f /var/swap
#加载sawp文件
$ swapon /var/swap
#关闭swap
$ swapoff /var/swap
#开机启动挂载
echo "/var/swap       swap     swap    defaults    0     0" >> "/etc/fstab"
```

