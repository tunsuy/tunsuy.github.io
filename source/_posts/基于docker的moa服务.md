---
title: 基于docker的moa服务
date: 2016-06-02 12:15:17
tags: [docker,linux]
categories: docker
---

# 一、 简介
该文章主要讲解将公司的server服务docker化的具体过程，从中可以看到很多docker相关知识的使用以及网络配置等操作  
文中还设计到自己实现的关于docker自动化配置的两个脚本，供大家学习参考

# 二、 docker化步骤
1、 将初始安装包拷贝到容器主机上

<!-- more -->

2、在主机上拉取一个centos6.7的docker镜像  
ps：因为官方的docker镜像都是轻量级的，所有很多linux下的命令都没有
* 方法一 就是要么自己将所有的命令工具都安装了
* 方法二 找一个别人建好的docker镜像

3、将主机上的安装包共享给容器，并交互式形式启动
```sh
docker run -t -i --name moa_centos7 -v /home/tunsuy/toContainer:/home/tunsuy/fromHost centos:centos7 /bin/bash
```
4、因为没有空白机的公有云安装包，所以这里以私有云空白安装包来安装  
ps：`  ./install.sh -m test -d public_clound  `
——参考./install.sh脚本实现

5、安装过程中可能会遇到一些问题  
* 提示没有xx命令——一律`  yum install xx `即可
* 提示连接不上mongodb  
——中断安装（其实moa依赖的一些软件环境已经安装成功了）  
——参考其他虚拟机公有云的mongodb安装情况，调整该docker容器中的mongodb并成功启动

6、单独安装公有云升级包（该包中需要包含dbserver服务包）

7、安装包成功之后，检查各项服务是否成功（包括web、流程、商店等）  
* 可能会发现mysql安装异常  
——手动安装：进入私有云空白安装包中的init目录下，运行./install.sh  
————安装过程中可能会报错，缺少某些依赖  
————依次yum install安装（有的可能需要自己下载安装）

8、所有的服务都检查无误时，制作镜像并上传
* 搭建私有仓库（按照官方文档做就可以了，这里不详细写）
* 将容器制作为镜像
  ```sh
  docker commit --author "ts" --message "更正moaserver初始化包环境" 容器名 docker.ts.com:5000/moaserver_env:init
  ```
* 上传镜像到私有仓库
  ```js
  docker push docker.ts.com:5000/moaserver_env:init
  ```
ps：配置私有仓库http访问  
编辑文件：` vim /etc/sysconfig/docker `  
添加以下内容
```sh
OPTIONS='--selinux-enabled --insecure-registry=docker.ts.com:5000'
```
或者编辑文件：` vim /usr/lib/systemd/system/docker.service `  
添加以下内容
```sh
  ExecStart=/usr/bin/docker-current daemon \
    --exec-opt native.cgroupdriver=systemd \
    $OPTIONS \
    $DOCKER_STORAGE_OPTIONS \
    $DOCKER_NETWORK_OPTIONS \
    $ADD_REGISTRY \
    $BLOCK_REGISTRY \
    $INSECURE_REGISTRY \
--insecure-registry docker.ts.com:5000
```

注：centos7的防火墙使用的是firewalld，因为之前对iptables比较熟，
     所以可以关掉firewalld，安装iptables服务  
```sh
systemctl stop firewalld
systemctl mask firewalld
yum install iptables-services
```
# 三、 服务使用
## 1、 从私有仓库拉取该镜像并使用
```js
    docker run -t -i --name=moaserver_env --net="none" -v /home/tunsuy/toContainer:/home/tunsuy/fromHost docker.ts.com:5000/moaserver_env:init /bin/bash
```
ps：这里采用none的网络方式，是因为想自己配置所有网络

## 2、 配置容器IP  
这里使用我写的一个工具脚本：
```js
sh bind_addr.sh moaserver_env 172.17.0.2  
```
ps：需要在主机上执行，而不是在容器中  
脚本地址：
[https://github.com/tunsuy/TSTools/tree/master/ShellTools/docker相关](https://github.com/tunsuy/TSTools/tree/master/ShellTools/docker相关)

## 3、 配置环境变量
```sh
source /etc/profile  
```
ps：由于之前在镜像中配置的环境变量，在容器中不能自动生效（原因不明）

## 4、 启动mongo和mysql
```sh
/etc/init.d/mongod start
/etc/init.d/mysqlaccd stop
/etc/init.d/mysqld start
```

## 5、 更新hosts文件
```sh
#将模板文件写入hosts：
cat /home/tunsuy/fromHost/host_templet >> /etc/hosts

#将hosts中的IP改成容器ip
mdbg -p 23808 -o changeip oldip=172.17.0.1,newip=172.17.0.2
```
ps：容器的/etc/hosts、/etc/resolv.conf文件是挂载在宿主机中的

## 6、 启动moa服务
```sh
/etc/init.d/moa start
```

## 7、端口映射
想要外部访问该docker，还需要进行主机和容器的端口映射  
有两种方式：
* 在新建一个容器的时候就指定（一般都是这样）
* 在已生成的容器中动态配置：主要还是基于iptables来设置（因为docker的原理实际上也是这样的）  
因为之前我们在创建容器的时候没有指定，所以这里以第二种来配置
```js
iptables -A DOCKER ! -i docker0 -o docker0 -p tcp --dport 443 -d 172.17.0.1 -j ACCEPT
iptables -t nat -A POSTROUTING -p tcp --dport 443 -s 172.17.0.1 -d 172.17.0.1 -j MASQUERADE
iptables -t nat -A DOCKER ! -i dokcer0 -p tcp --dport 443 -j DNAT --to-destination 172.17.0.1:443
```
ps：为了方便，写了一个工具脚本
```js
sh docker_expose.sh moaserver_env 172.17.0.2 add 4432:443
```
脚本地址：
        [https://github.com/tunsuy/TSTools/tree/master/ShellTools/docker相关](https://github.com/tunsuy/TSTools/tree/master/ShellTools/docker相关)

注：查看iptables情况
```js
iptables -nvxL --line-numbers
iptables -t nat -nvxL --line-numbers
```

# 四、 不足之处
因为所有的容器都是基于宿主机的，最多通过修改时区来设置，也就只能偏移24小时  
网上有一种方案，不知可行不：https://github.com/wolfcw/libfaketime/

# 五、 参考链接
https://yeasy.gitbooks.io/docker_practice/content/cases/supervisor.html
