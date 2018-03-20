---
layout: post
author: sjf0115
title: Linux rsync命令使用指南
date: 2018-03-20 15:17:17
tags:
  - Linux
  - Linux 命令

categories: Linux
permalink: rsync-command-usage
---

### 1. 概述

`rsync` 命令是一个远程数据同步工具，可通过 LAN/WAN 快速同步多台主机间的文件。`rsync` 使用所谓的 "rsync算法" 来使本地和远程两个主机之间的文件达到同步，这个算法只传送两个文件的不同部分，而不是每次都整份传送，因此速度相当快。`rsync` 是一个功能非常强大的工具，其命令也有很多功能特色选项。

### 2. 安装

在 ubuntu 下安装 rsync 通过以步骤可以实现：
```
sudo apt-get install rsync xinetd
```
默认情况下 ubuntu 安装了 rsync，因此只需安装 xinetd即可:
```
sudo apt-get install xinetd
```
### 3. 配置

编辑 `/etc/default/rsync` 启动 rsync 作为使用 xinetd 的守护进程：
```
# 打开rsync
sudo vim /etc/default/rsync
# 编辑rsync
RSYNC_ENABLE=inetd
```

```
xiaosi@ying:/etc/apt$ rsync test@123.206.187.64::ubuntu
rsync: failed to connect to 123.206.187.64 (123.206.187.64): Connection refused (111)
rsync error: error in socket IO (code 10) at clientserver.c(128) [Receiver=3.1.1]
```
查看服务是否启动：
```
ubuntu@VM-0-7-ubuntu:~$  ps -ef | grep rsync
root     18848     1  0 17:29 ?        00:00:00 rsync --daemon --config=/etc/rsyncd.conf
ubuntu   18850 12214  0 17:29 pts/0    00:00:00 grep --color=auto rsync
```

https://segmentfault.com/a/1190000010310496

### 2. 语法
```
rsync [OPTION...] SRC... [DEST]

rsync [OPTION...] [USER@]HOST:SRC... [DEST]

rsync [OPTION...] SRC... [USER@]HOST:DEST

rsync [OPTION...] [USER@]HOST::SRC... [DEST]
rsync [OPTION...] rsync://[USER@]HOST[:PORT]/SRC... [DEST]

rsync [OPTION...] SRC... [USER@]HOST::DEST
rsync [OPTION...] SRC... rsync://[USER@]HOST[:PORT]/DEST
```

#### 2.1 本地模式

```
rsync [OPTION...] [USER@]HOST:SRC... [DEST]
```
拷贝本地文件。当 SRC 和 DEST 路径信息都不包含有单个冒号 `:` 分隔符时就启动这种工作模式。例如：
```
xiaosi@ying:~$ rsync -a adv_push adv_push_backup/
xiaosi@ying:~$ ll adv_push_backup/
总用量 20
drwxrwxr-x  3 xiaosi xiaosi  4096  3月 20 15:29 ./
drwxr-xr-x 83 xiaosi xiaosi 12288  3月 20 15:29 ../
drwxrwxr-x  2 xiaosi xiaosi  4096  3月 20 15:28 adv_push/
```

#### 2.2 通过远程Shell访问-Pull

```
rsync [OPTION...] [USER@]HOST:SRC... [DEST]
```
使用一个远程Shell程序(如rsh、ssh)来实现将远程机器的内容拷贝到本地机器。当 SRC 地址路径包含单个冒号 `:` 分隔符时启动该模式。例如：
```

```

#### 2.3 通过远程Shell访问-Push

```
rsync [OPTION...] SRC... [USER@]HOST:DEST
```
使用一个远程shell程序(如rsh、ssh)来实现将本地机器的内容拷贝到远程机器。当 DEST 路径地址包含单个冒号 `:` 分隔符时启动该模式。例如：
```
```

#### 2.4 通过rsync进程访问-Pull

```
rsync [OPTION...] [USER@]HOST::SRC... [DEST]
rsync [OPTION...] rsync://[USER@]HOST[:PORT]/SRC... [DEST]
```
从远程 rsync 服务器中拷贝文件到本地机。当 SRC 路径信息包含 `::` 分隔符时启动该模式。例如：

#### 2.5 通过rsync进程访问-Push

```
rsync [OPTION...] SRC... [USER@]HOST::DEST
rsync [OPTION...] SRC... rsync://[USER@]HOST[:PORT]/DEST
```
从本地机器拷贝文件到远程 rsync 服务器中。当 DEST 路径信息包含 `::` 分隔符时启动该模式。例如：

#### 2.6 查阅模式

只使用一个 SRC 参数，而不使用 DEST 参数将列出源文件而不是进行复制。例如：
```
xiaosi@ying:~$ rsync -a adv_push
drwxrwxr-x          4,096 2018/03/20 15:28:15 adv_push
-rw-rw-r--            353 2018/03/14 17:02:35 adv_push/adv_push_20180307.txt
-rw-rw-r--            353 2018/03/14 17:02:33 adv_push/adv_push_20180308.txt
```





































.....
