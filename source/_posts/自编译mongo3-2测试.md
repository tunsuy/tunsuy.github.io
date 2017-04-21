---
title: 自编译mongo3.2测试
date: 2016-12-09 10:13:44
tags: [mongodb,linux]
categories: mongodb
---

# 一、 简介
公司将mongodb删除机制进行了自定义，改动了相关源码，故需要配合测试mongodb的功能是否正常

# 二、 检查服务器
## 1、 检查mongo启动情况
```js
    netstat -anpt | grep mong* | grep 0.0.0.0
    mongoa --port 27020（相应的端口）——进入mongo
```

<!-- more -->

## 2、 集群数据库情况
`cat /etc/hosts | grep db`
——3个数据库分片（sh0，sh1，sh2）  
——2个统计分片（statis-sh0，statis-sh1）

也可以这样查看：
* 进入mongos（169.100下）
* 进入config数据库中
```js
        db.shards.find()
```
# 三、 主备切换测试
## 1、 shard0：
* 随便进入一个sh0节点服务器  
通过/etc/hosts可以查看到（169.100下）
* 查看sh0集群情况： `rs.status()`
找到sh0主节点服务器，并进入, 执行`rs.stepDown()`——切换主备
* 检查主备切换是否成功`rs.status()`——集群情况

## 2、 依次对shard1、shard2集群进行主备切换

也可以关注下主备切换的日志情况
```js
    find / -name mongo*.log（找到各mongo进程log）
```

## 四、 chunk迁移测试
（以迁移customer.info表为例）
## 1、 查看迁移之前的chunk情况
```js
    use config
    db.chunks.find({"_id":/customer.info-did_10000*/, "ns":"customer.info"}).pretty()
    {
        "_id" : "customer.info-did_10000custmid_100",
        "lastmod" : Timestamp(135, 0),
        "lastmodEpoch" : ObjectId("54ae3eaaf3a2f2e7d3726927"),
        "ns" : "customer.info",
        "min" : {
            "did" : NumberLong(10000),
            "custmid" : NumberLong(100)
        },
        "max" : {
            "did" : NumberLong(10000),
            "custmid" : NumberLong(1948)
        },
        "shard" : "shard0"
    }
    {
        "_id" : "customer.info-did_10000custmid_1948",
        "lastmod" : Timestamp(136, 0),
        "lastmodEpoch" : ObjectId("54ae3eaaf3a2f2e7d3726927"),
        "ns" : "customer.info",
        "min" : {
            "did" : NumberLong(10000),
            "custmid" : NumberLong(1948)
        },
        "max" : {
            "did" : NumberLong(10039),
            "custmid" : NumberLong(451)
        },
        "shard" : "shard2"
    }
```

## 2、 执行迁移命令
```js
    sh.moveChunk("customer.info", {did:10000,custmid:100}, "shard1")
#或者
    db.adminCommand({moveChunk:"customer.info", find:{did:10000,custmid:100}, to:"shard1"})
```
PS：一般执行命令都有这两种方式
* sh. 方式可以不用切换数据库执行
* db. 方式必须在admin数据库下
PS：迁移条件必须包含片键

## 3、 查看迁移情况
```js
    use config
    db.chunks.find({"_id":/customer.info-did_10000*/, "ns":"customer.info"}).pretty()
```

也可以关注下主备切换的日志情况
```js
    find / -name mongo*.log（找到各mongo进程log）
```

# 五、 chunk分裂
## 1、 查看chunk分裂前情况
```js
    use config
    db.chunks.find({"_id":/customer.info-did_10000*/, "ns":"customer.info"}).pretty()
```

## 2、 指定分裂点
```js
    sh.splitAt("customer.info", {"did":10000, "custmid":500})
```

## 3、 检查分裂情况
```js
    db.chunks.find({"_id":/customer.info-did_10000*/, "ns":"customer.info"}).pretty()
```

# 六、 其他
## 1、 重启moa服务  
PS：因为有的服务可能有缓存，所有在操作了数据库之后需要重启服务才能重新读取数据库数据

## 2、 检查各模块功能情况  
PS：因为涉及到php、go、c的mongo库，所以需要覆盖以下测试
* web端、PC端、运营、统计
* 主要关注点  
——涉及数据库读写操作的功能  
——大量读写操作情况  
——导入导出情况  
——私有云服务器接入情况  
——数据删除情况  
——数据恢复情况

## 3、 检查服务器资源情况  
PS：可以写一个小的脚本（超过一定的阈值则输出到log里面），然后一直在后台运行，然后操作业务  
最后测试完之后，检查log文件中的资源记录情况
