---
title: Linux定时任务（crontab）
tags:
  - linux
categories:
  - 服务器
  - linux指令
abbrlink: 34177
date: 2018-06-16 18:55:53
---

## 1、什么是例行性工作调度
>每个人或多或少都有一些约会或者是工作，有的工作是例行性的， 例如每年一次的加薪、每 个月一次的工作报告、每周一次的午餐会报、每天需要的打卡等等； 有的工作则是临时发生 的，例如刚好总公司有高官来访，需要你准备演讲器材等等！ 用在生活上面，例如每年的爱 人的生日、每天的起床时间等等、还有突发性的 3C 用品大降价 （啊！真希望天天都有！） 等等啰。
像上面这些例行性工作，通常你得要记录在行事历上面才能避免忘记！不过，由于我们常常 在计算机前面的缘故， 如果计算机系统能够主动的通知我们的话，那么不就轻松多了！嘿 嘿！这个时候 Linux 的例行性工作调度就可以派上场了！ 在不考虑硬件与我们服务器的链接 状态下，我们的 Linux 可以帮你提醒很多任务，例如：每一天早上 8:00 钟要服务器连接上音 响，并启动音乐来唤你起床；而中午 12:00 希望 Linux 可以发一封信到你的邮件信箱，提醒 你可以去吃午餐了； 另外，在每年的你爱人生日的前一天，先发封信提醒你，以免忘记这么 重要的一天。
那么 Linux 的例行性工作是如何进行调度的呢？所谓的调度就是将这些工作安排执行的流程 之意！ 咱们的 Linux 调度就是通过 crontab 与 at 这两个东西！这两个玩意儿有啥异同？就让 我们来瞧瞧先！

## 2、crond 服务读取配置文件的位置 一般来说，crond 默认有三个地方会有执行脚本配置文件，他们分别是：
* /etc/crontab
* /etc/cron.d/* 
* /var/spool/cron/*
>这三个地方中，跟系统的运行比较有关系的两个配置文件是放在 /etc/crontab 文件内以及 /etc/cron.d/* 目录内的文件， 另外一个是跟用户自己的工作比较有关的配置文件，就是放在 /var/spool/cron/ 里面的文件群。

## 3、crontab 简介
crontab ：crontab 这个指令所设置的工作将会循环的一直进行下去！
可循环的时间为分钟、小时、每周、每月或每年等。crontab 除了可以使用指令执行外，亦可编辑 /etc/crontab 来支持。 至于让 crontab 可以生效的服务则是 crond 这个服务喔！

crond是linux下用来周期性的执行某种任务或等待处理某些事件的一个守护进程，与windows下的计划任务类似，当安装完成操作系统后，默认会安装此服务工具，并且会自动启动crond进程，crond进程每分钟会定期检查是否有要执行的任务，如果有要执行的任务，则自动执行该任务。

Linux下的任务调度分为两类，系统任务调度和用户任务调度。
系统任务调度：系统周期性所要执行的工作，比如写缓存数据到硬盘、日志清理等。在/etc目录下有一个crontab文件，这个就是系统任务调度的配置文件。
/etc/crontab文件包括下面几行(里面的内容下面讲解)：

```
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user	command
17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
```
用户任务调度：用户定期要执行的工作，比如用户数据备份、定时邮件提醒等。用户可以使用 crontab 工具来定制自己的计划任务。所有用户定义的crontab 文件都被保存在 /var/spool/cron目录中。其文件名与用户名一致。

### 上面时间格式解释
![](http://image.sweetm.top//18-6-10/11687506.jpg)

## cron控制
>安装crontab：
yum install crontabs
服务操作说明：
/sbin/service crond start //启动服务
/sbin/service crond stop //关闭服务
/sbin/service crond restart //重启服务
/sbin/service crond reload //重新载入配置
查看crontab服务状态：
service crond status
手动启动crontab服务：
service crond start

