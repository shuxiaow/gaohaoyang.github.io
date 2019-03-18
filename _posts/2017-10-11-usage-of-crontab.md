---
layout: post
title:  "使用crontab设置定时任务"
categories: linux 
tags: crontab 
author: ShuxiaoW
---

* content
{:toc}

本文介绍如何在Linux上设置定时任务

## Step 1 设置好正确的时区与时间

Linux下我们可以针对每个user设置时区和crontab任务。但是，不管是哪个user，cron在执行定时任务时，总是按照cron守护进程获取到的系统时间和时区为准的。

> Notice that tasks will be started based on the cron's system daemon's notion of time and timezones.

所以，我们首先需要确定系统当前的时区设置跟你期望的时间是一致的。如果系统时区不对，可以这样设置：

```sh
$ sudo dpkg-reconfigure tzdata
$ sudo service cron restart
```

## Step 2 设置定时任务

有些任务需要用root权限来执行，有些只需要用普通user用户来执行。建议是除非必须要用root，否则一律用普通user吧。

```sh
# 编辑当前登录user的crontab任务
$ crontab -e
```

按照如下格式在打开的文件里面添加任务即可，每个任务只能保存为一行。

```sh
# m h  dom mon dow   command
```

添加完成后，可以查看是否设置成功

```sh
$ crontab -l
```

## Step 3 检查是否执行成功

ubuntu系统中，crontab任务执行的结果是打印到`/var/log/syslog`的

```sh
$ grep CRON /var/log/syslog
```
