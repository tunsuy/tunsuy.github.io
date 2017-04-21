---
title: calabash-android原理分析
date: 2016-06-20 15:16:39
tags: [android,自动化测试]
categories: android
---

在前面已经简单的介绍了calabash-ios的原理，这里将继续分析calabash-Android端的原理

# 一、 原理解析
calabash-android架构其实与IOS是相同的

1、内部使用核心为cucumber的calabash的脚本在运行测试的时候会在虚拟机/真机上预装一个web服务器，这个web服务器就是解释calabash的脚本，

2、因为calabash-Android是基于robotium框架的，所以在机器上预装的web-server会将下发下来的calabash脚本解释为robotium的脚本，然后向测试app发送robotium的脚本，

<!-- more -->

3、因为robotium框架就是封装的google测试框架instumentation，所以app拿到robotium脚本后，将其解释为instumentation命令向被测试的app发送这些命令，被测试的app执行这些命令，然后将结果返回。

# 二、 系统架构图
calabash-Android整个框架采用 C/S 的运行模式，系统架构如下图所示：

{% asset_img calabash-android.jpg %}


# 三、 框架图解释

1、Runner 负责接受用户指令，并对其进行数据校验、指令转换等操作，之后将其交由客户端处理（这里的客户端是指运行在pc上的用户自己编写的代码），

2、instrumentation test server就是预装在设备上的web-server，客户端在接收到指令之后，将指令发送给它，请求其执行对应操作，

3、web-server在接收到指令之后，解释该命令并与app进行交互，

4、所有执行结果最终会被收集到 Results 中。
