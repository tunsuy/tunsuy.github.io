---
title: docker自定义daemon选项
date: 2016-06-15 12:10:23
tags: [docker,linux]
categories: docker
---

# 一、 前提
可以定义的选项为如下列出来的
```js
    docker-current daemon --help
```

# 二、 方案一
## 1、 分别定义各环境文件
```js
    eg：/etc/sysconfig/docker-storage
    /etc/sysconfig/docker-network
```
该文件中可以自定义环境变量

<!-- more -->

## 2、 引入环境文件
在`/usr/lib/systemd/system/docker.service`文件中引入环境文件
```js
    eg：EnvironmentFile=-/etc/sysconfig/docker-storage
    EnvironmentFile=-/etc/sysconfig/docker-network
```
并使用其环境变量
```js
    ExecStart=/usr/bin/docker-current daemon \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
```
备注：也可以将所有的自定义选项定义在一个文件中

例：将官方镜像仓库替换为阿里云镜像仓库
* 前提：注册阿里云用户，此时才会获取到一个私有的镜像仓库地址
          https://cr.console.aliyun.com/
* 在`/etc/sysconfig/docker`中的option变量中加上如下选项
```js
        --registry-mirror=https://u6nqa61i.mirror.aliyuncs.com
```
* 重启docker：
```js
        systemctl restart docker
```

# 三、 方案二
直接在`/usr/lib/systemd/system/docker.service`文件中加入选项
```js
    eg：ExecStart=/usr/bin/docker-current daemon \
          --exec-opt native.cgroupdriver=systemd \
          --insecure-registry docker.ts.com:5000
```

# 四、 方案三
使用`daemon.json`文件  
官方文档：  
https://docs.docker.com/engine/reference/commandline/dockerd/#/linux-configuration-file  
路径：`/etc/docker/daemon.json`  
备注：使用此方案不要与其他方案相混合使用，否则docker启动的时候会报错
