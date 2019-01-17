---
layout:     post
title:      "本地安装多个mysql，配置主从复制"
subtitle:   "免安装mysql采坑"
date:       2019-01-19
author:     "Hwt"
header-img: "img/post-bg-js-version.jpg"
tags:
    - 笔记
---

今天下午配置主从配置，配置失败。从库能接收到主库的日志，但是从库却不执行日志里面的语句。仔细检查了配置的过程与错误日志，都没有发现问题。然后就想到可能是从库的问题，因为从库是自己在使用wamp时一起安装的，版本为5.6(主库为5.7)。


所以打算另外安装一个5.7版本的mysql测试。为了方便直接下载是免安装版，下面介绍一下步骤

### 1. 下载
[下载地址](https://dev.mysql.com/downloads/mysql/5.7.html#downloads)，解压即可
![mysql](/img/mysql.png)
### 2. 书写配置文件
在解压目录下新建文件`my.ini`，写入一下内容
```text
[mysql]
default-character-set=utf8

[mysqld]
# 3306已经被使用了
port = 3305

basedir=D:/soft_install/mysql-5.7.24-winx64

# 数据库的放置文件夹
datadir=D:/soft_install/mysql-5.7.24-winx64/data

max_connections=200
character-set-server=utf8
default-storage-engine=INNODB
```

### 3. 安装
进入解压目录的bin
```text
D:\soft_install\mysql-5.7.24-winx64\bin>mysqld --initialize
```
等待执行完成，这一步是生成基本的数据库以及其他文件。如果出现错误可以在bin文件夹下查看`.err`结尾日志文件。

双击mysqld.exe启动
### 4. 登录
如果空密码不能登录，可以在`.err`结尾文件中找到初始密码

```text
[Note] A temporary password is generated for root@localhost: rAhBelIR5e!?
```


### 5. 注册服务
在bin目录下
```text
D:\soft_install\mysql-5.7.24-winx64\bin>mysqld install mysql57_2
Service successfully installed.
```
执行下面语句启动
```text
net start mysql57_2
```

> 不知道为什么我注册的服务无法启动，还没有找到原因

使用这个新安装的数据库就可以整个主从复制可以正常运行，说明真的是数据库的问题。