title: Nginx日志定时切割
author: bliver
tags:
  - Nginx
categories:
  - 技术
date: 2016-04-23 15:27:57
---
Nginx本身不支持日志的自动切割，需要通过其它工具协助来完成，下面记录了两种方法。推荐Logrotate，因为它可定制的地方比较多，配置起来也方便，不需要自己写脚本，不容易出错。另外通过 yum 或 apt 等方式安装新版的 Nginx 会自动创建Logrotate的运行脚本，只需要开启定时就可以了。如果没有，也可以通过下面的内容自己创建。

<!-- more -->

## Logrotate大法：

Logrotate是基于CRON来运行的，相关的脚本和配置文件信息如下：

* crontab配置：`crontab -e`，其中主要用到的是`0 0 * * * root run-parts /etc/cron.daily`，因为Logrotate的运行脚本在这个目录下。
	
	```
	# 指定 cron 执行脚本时的 shell 环境
	SHELL=/bin/bash
	# 设置 PATH，防止运行脚本时可能找不到命令
	PATH=/sbin:/bin:/usr/sbin:/usr/bin
	# 收件人，为空表示不发送给任何用户
	MAILTO=""
	# 执行脚本时的主目录
	HOME=/
	
	# run-parts
	0 * * * * root run-parts /etc/cron.hourly
	0 0 * * * root run-parts /etc/cron.daily
	0 0 * * 0 root run-parts /etc/cron.weekly
	0 0 1 * * root run-parts /etc/cron.monthly
	```

* Logrotate脚本：`/etc/cron.daily/logrotate`，这个脚本会调用`/etc/logrotate.conf`配置文件。

	```
	#!/bin/sh

	/usr/sbin/logrotate /etc/logrotate.conf >/dev/null 2>&1
	EXITVALUE=$?
	if [ $EXITVALUE != 0 ]; then
	    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
	fi
	exit 0
	```
	
* Logrotate配置文件：`/etc/logrotate.conf`，可以看做是Logrotate配置的默认值，可以通过`/etc/logrotate.d`目录下的配置覆盖。

	```
	# see "man logrotate" for details
	# rotate log files weekly
	weekly
	
	# keep 4 weeks worth of backlogs
	rotate 4
	
	# create new (empty) log files after rotating old ones
	create
	
	# use date as a suffix of the rotated file
	dateext
	
	# uncomment this if you want your log files compressed
	#compress
	
	# RPM packages drop log rotation information into this directory
	include /etc/logrotate.d
	
	# no packages own wtmp and btmp -- we'll rotate them here
	/var/log/wtmp {
	    monthly
	    create 0664 root utmp
		minsize 1M
	    rotate 1
	}
	
	/var/log/btmp {
	    missingok
	    monthly
	    create 0600 root utmp
	    rotate 1
	}
	
	# system-specific logs may be also be configured here.
	```
	
* `/etc/logrotate.d`下面的所有文件，主要介绍 nginx：

	```
	/var/log/nginx/*.log {
        daily
        missingok
        rotate 52
        compress
        delaycompress
        notifempty
        create 640 nginx adm
        dateext
        sharedscripts
        postrotate
                [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
        endscript
}
	```

## 自定义脚本

思路很简单，主要就是创建一个脚本，根据指定格式切割并转存日志，然后通过 nginx 命令创建新的日志文件。以下是引用自[博客](http://blog.sina.com.cn/s/blog_5f54f0be0100zaza.html):

创建日志分割脚本

1. 登录SSH，创建cut_logs.sh文件

	```
	vi /root/cut_logs.sh
	```
	
2. 粘贴下面代码到cut_logs.sh，并保存

	```
	#!/bin/bash
	# The Nginx logs path
	logs_path="/home/wwwlogs/"
	mkdir -p ${logs_path}$(date -d "yesterday" +"%Y")/$(date -d "yesterday" +"%m")/
	mv ${logs_path}www.juzihc.com.log ${logs_path}$(date -d "yesterday" +"%Y")/$(date -d "yesterday" +"%m")/juzihc_$(date -d "yesterday" +"%Y%m%d").log
	kill -USR1 $(cat /usr/local/nginx/logs/nginx.pid)
	```
3. 添加cut_logs.sh执行权限

	```
	chmod +x /root/cut_logs.sh
	```
4. 设置cut_logs.sh启动时间,执行命令crontab -e进入编辑状态,添加如下代码,每天0点01分启动。
	
	```
	01 00 * * * /root/cut_logs.sh
	```

## 附录：
logrotate 配置文件的主要参数：

* `daily`：指定转储周期为每天 
* `weekly`：指定转储周期为每周 
* `monthly`：指定转储周期为每月 
* `dateext`：在文件末尾添加当前日期 
* `compress`：通过gzip 压缩转储以后的日志 
* `nocompress`：不需要压缩时，用这个参数 
* `copytruncate`：先把日志内容复制到旧日志文件后才清除日志文件内* `容，可以保证日志记录的连续性
* `nocopytruncate`：备份日志文件但是不截断 
* `create mode owner group`：转储文件，使用指定的文件模式创建新的日志文件 
* `nocreate`：不建立新的日志文件 
* `delaycompress`：和 compress 一起使用时，转储的日志文件到下一次转储时才压缩 
* `nodelaycompress`：覆盖 delaycompress 选项，转储同时压缩。 
* `errors address`：专储时的错误信息发送到指定的Email 地址 
* `ifempty`：即使是空文件也转储，这个是 logrotate 的缺省选项。 
* `notifempty`：如果是空文件的话，不转储 
* `mail address`：把转储的日志文件发送到指定的E-mail 地址 
* `nomail转储时不发送日志文件 
* `olddir directory`：转储后的日志文件放入指定的目录，必须和当前日志文件在同一个文件系统 
* `noolddir`：转储后的日志文件和当前日志文件放在同一个目录下 
* `rotate count`：指定日志文件删除之前转储的次数，0 指没有备份，5 指保留5 个备份 
* `tabootext [+] list`：让logrotate 不转储指定扩展名的文件，缺省的扩展名是：.rpm-orig, .rpmsave, v, 和 ~ 
* `size size`：当日志文件到达指定的大小时才转储，Size 可以指定 bytes (缺省)以及KB (sizek)或者MB (sizem). 
* `prerotate/endscript`：在转储以前需要执行的命令可以放入这个对，这两个关键字必须单独成行
* `postrotate/endscript`：在转储以后需要执行的命令可以放入这个对，这两个关键字必须单独成行