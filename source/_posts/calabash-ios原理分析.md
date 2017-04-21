---
title: calabash-ios原理分析
date: 2015-06-12 15:10:51
tags: [ios, 自动化测试]
categories: ios
---

calabash同时支持ios和Android平台，这里只介绍ios平台。

# 一、 系统架构图
calabash-ios整个框架采用 C/S 的运行模式，系统架构如下图所示：

<!-- more -->

{% asset_img calabash-ios.jpg %}


# 二、 框架分析
接上图

1、其中，Runner 负责接受用户指令，并对其进行数据校验、指令转换等操作

2、之后将其交给客户端处理（这里的客户端是指运行pc上的代码）

3、客户端将指令发送给对应服务器http server（这里的服务器就是编译进app中的calabash.framework），请求其执行对应操作

4、client-server两者之间通过 JSON 格式的数据进行交互

5、所有执行结果最终会被收集到 Results 中。

6、Calabash-iOS 服务器是基于Frank构建的，Frank也是一种基于cucumber的自动化测试框架。
