---
title: App三方交互逻辑实践
date: 2016-11-16 15:28:06
tags: [ios,go,服务端]
categories: ios
---

项目地址：  
[https://github.com/tunsuy/AppInteractive](https://github.com/tunsuy/AppInteractive)

# 一、 背景介绍
之前有一个公司项目与第三方应用交互的逻辑需要全面测试，那么怎样才能测试到整个交互的所有环节呢

那就是需要一个实际的第三方，并且这个第三方需要包括server端和app端

<!-- more -->

既然是测试，那么我们只需要将公司项目跟三方有交互的接口做个全面的测试即可，不在乎这个三方是怎样的形式

那么这就好办了，自己实现一个这个的三方项目

# 二、 交互逻辑图
1、口袋助理跳转第三方app逻辑示意图：

{% asset_img moa2app.jpg %}

2、第三方跳转口袋助理逻辑示意图：

{% asset_img app2moa.jpg %}

# 三、 实现
server端：go语言实现  
app端：iOS

具体实现见github项目地址

根据相关协议从代码层面详细的测试每个交互接口：
* 兼容性
* 容错性
* 健壮性
