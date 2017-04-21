---
title: centos下安装mongodb记录
date: 2015-03-10 19:26:20
tags: [linux,mongdb]
categories: mongodb
---

# 一、 安装步骤
## 1、 卸载已有mongodb数据库
```js
    /etc/init.d/mongod stop
    yum erase $(rpm -qa | grep mongodb-org)
```
删除mongodb相关目录文件等，例如：
```js
    rm -r /var/log/mongodb/
    rm -r /var/lib/mongo
```

<!-- more -->

## 2、 下载最新mongodb
```js
    wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel62-3.2.9.tgz
```

## 3、 移动下载文件并解压
```js
    mv ./mongodb-linux-x86_64-rhel62-3.2.9.tgz /usr/local/src/
    tar -xvf mongodb-linux-x86_64-rhel62-3.2.9.tgz
```

## 4、 移动解压文件
```js
    mv ./mongodb-linux-x86_64-rhel62-3.2.9 /usr/local/mongodb
```

## 5、 创建mongodb相关目录
```js
    mkdir -p /home/ts/db/mongodb/data
    mkdir -p /home/ts/db/mongodb/log
```

## 6、 创建并编辑mongodb配置文件
```js
    vim /home/ts/db/mongodb/mongodb.conf
```

# 二、 配置mongodb环境变量
编辑：`vim /etc/profile`  
加入如下语句：
```js
    # mongodb set
    export MONGODB=/usr/local/mongodb
    PATH=$PATH:$MONGODB/bin
```
立即生效：
```js
    source /etc/profile
```

# 三、 配置开机启动
编辑：`vim /etc/rc.d/init.d/mongod`  
注：必须在该文件开头写上：       
```js
    # chkconfig: 2345 10 90
    # description: mongod ....
```
设置：
```js
    chmod +x /etc/rc.d/init.d/mongod
    chkconfig mongod on
```

PS：`/etc/init.d/` 是 `/etc/rc.d/init.d/` 的软链接
