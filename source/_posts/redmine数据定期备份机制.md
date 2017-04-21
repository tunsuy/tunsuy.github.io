---
title: redmine数据定期备份机制
date: 2015-03-20 15:21:24
tags: [redmine,linux]
categories: redmine
keywords: [redmine,linux,备份,cron]
description:
---

# 一、 简介
目前项目管理用的是redmine，上面有bug记录和功能记录，为了防止因为某些原因导致数据不可恢复性的丢失，故需要对redmine进行定期的备份。

虽然redmine使用的是mysql数据库，可以使用mysql自带的工具进行数据的备份恢复，但是因为redmine上的数据包括各种附件（图片和文件等），所以如果是只是备份mysql数据库的话，那么这些附件必然会丢失。所以决定采用整个项目备份的方案：将整个项目每天定时压缩，按日期来命名，然后上传到一个专门的备份服务器上保存。同时该服务器上每天定时清理7天之前的备份数据，避免备份数据无限制增长。

<!-- more -->

# 二、 备份机制
备份脚本存放在redmine系统上，每天凌晨2点左右备份，并上传到备份服务器，全量备份：备份的是整个redmine项目，以时间命名文件，可以看到最新备份的是哪一天的文件。  
具体脚本如下：
```sh
#!/bin/bash
#program:
#       usage:sh backup_redmine.sh
#       function:auto backup redmine to IP at 2 o'clock everyday
#author：tunsuy
#date：2014/8/27
#last change date:2014/8/27

PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:~/bin
export PATH

function get_shell_dir() {
	SOURCE="$0"

	while [ -h "$SOURCE"  ]; do
		DIR="$( cd -P "$( dirname "$SOURCE"  )" && pwd  )"
		SOURCE="$(readlink "$SOURCE")"
		[[ $SOURCE != /*  ]] && SOURCE="$DIR/$SOURCE"
	done

	DIR="$( cd -P "$( dirname "$SOURCE"  )" && pwd  )"
	echo $DIR
}

dir=$(get_shell_dir)

DATE=$(date +"%Y-%m-%d")

tar -czf ${dir}/redmine_backup_${DATE}.tar.gz /opt/redmine-2.6.0-3

password="moatest"

/bin/rpm -qa | grep -q expect

if [ $? -ne 0 ]; then
	echo "please install expect"
	exit 1
fi

expect -c "
	spawn scp -P 22 ${dir}/redmine_backup_${DATE}.tar.gz root@200.200.169.161:/home/tunsuy/redmine_backup;
	expect {
		\"yes/no\" {send \"yes\r\"; exp_continue;}
		\"password\" {set timeout 300; send \"$password\r\";}
	}
expect eof"

rm -rf ${dir}/redmine_backup_${DATE}.tar.gz
```
采用的是linux下cron定时机制，定时配置脚本如下：
```sh
59 1 * * * /home/ts/backup_redmine.sh >/home/ts/backup-redmine.log 2>&1
```
脚本执行log：/home/ts/backup-redmine.log

# 三、 删除备份文件机制
为了防止备份服务器上redmine的备份文件无限制增长，故每天凌晨3点左右，执行删除备份文件脚本，删除7天前的备份  
删除备份脚本如下：
```sh
#!/bin/bash

find /home/tunsuy/redmine_backup/ -mtime +7 -name "redmine_backup_*.tar.gz" -exec rm -rf {} \;
```
同样的，采用的是linux下cron定时机制，定时配置脚本如下：
```sh
00 3 * * * /home/tunsuy/del-7days-ago-redmine.sh >/home/tunsuy/del-redmine.log 2>&1
```
脚本执行log：/home/tunsuy/del-redmine.log

# 四、 cron简介
1、cron定时执行日志：/var/log/cron*  

2、cron定时配置文件：/var/spool/cron/root  
注：通过cron -e命令会自动打开一个文件，然后将定时任务写进去，保存，将在/var/spool/cron/下生成一个当前用户名命名的文件

3、cron -l可以查看所有的定时任务

