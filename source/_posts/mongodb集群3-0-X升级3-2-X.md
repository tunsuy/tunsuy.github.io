---
title: mongodb集群3.0.X升级3.2.X
date: 2016-10-15 10:07:58
tags: [mongodb,linux]
categories: mongodb
---

# 一、 简介
该文章主要根据官方英文文档来操作的

官方文档：https://docs.mongodb.com/manual/release-notes/3.2-upgrade/

# 二、 升级准备工作
1、下载二进制文件  
```js
    wget https://www.mongodb.com/dr/fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel62-3.2.10.tgz
```
2、将该文件放入所有的节点服务器下并解压
```js
    tar -zxvf XXXX
```

<!-- more -->

3、备份configdb（可选）  
在每个configdb服务器执行：
```js
    cp -r /home/moa/db/configdb/data/ /home/moa/db/configdb/data_bak/
```
4、关闭均衡器
* 查看均衡器状态: `sh.getBalancerState()`
* 关闭: `sh.stopBalancer()`
* 检查数据没有在迁移
```js
        use config
        while( sh.isBalancerRunning() ) {
            print("waiting...");
            sleep(1000);
        }
```

# 三、 开始升级  
## 1、 首先升级分片0  
1.1 升级分片0集群中的secondary节点
* 停掉该mongod0进程—kill pid
* 使用最新mongodb进程替换掉mongod0  
```js
        #备份：
        mv /usr/local/mongodb/bin/mongod0 /usr/local/mongodb/bin/mongod0.bak  

        #替换：
        cp /home/mongodb-linux-x86_64-rhel62-3.2.10/bin/mongod /usr/local/mongodb/bin/mongod0
```
* 启动该mongod0进程—`/etc/init.d/mongod0 restart`

按照1.1方法依次升级其他的secondary节点  
ps：在确认上一个secondary状态正常的情况再进行: `rs.status()`

1.2 升级分片0集群中的primary节点
* 停掉primary—`rs.stepDown()`
* 检查是否产生了新的primary—`rs.status()`  
ps：在其他两个节点查看
* 按照1的方法升级其他分片

1.3 更新选举协议
在primary节点执行以下操作：
```js
    cfg=rs.conf();
    cfg.protocolVersion=1;
    rs.reconfig(cfg);
```

## 2、 升级分片1  
按照1中的方法进行

## 3、 升级config 服务器
* 顺序：需要按照mongos中配置的逆序升级
```js
        cat /home/moa/db/mongos/mongos.conf
        ——configdb=config0.moadb.com,config1.moadb.com,config2.moadb.com
```
所以升级顺序应该是：config2、config1、config0

* 升级：按照1中方法进行（只是这里变成了mongocfgd）

## 4、 升级mognos服务器  
按照1中方法进行（只是这样换成了mongos）

## 5、 替换掉所以的mongo工具（可选）
```js
    cp ./mongodb-linux-x86_64-rhel62-3.2.10/bin/* /usr/local/mongodb/bin/
```
