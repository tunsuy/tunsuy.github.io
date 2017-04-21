---
title: calabash-ios安装详解
date: 2016-06-28 14:48:31
tags: [ios,自动化测试]
categories: ios
---

# 一、 简介
为了更好的理解业内app自动化测试框架的原理机制，以便看是否有很好的办法在解决目前项目自动化测试存在的一些问题  

这次先介绍calabash的使用过程，后续有时间会看下其源码实现机制。

<!-- more -->

# 二、 工具安装
这里以我自身的安装成功经历记录如下：  
1、启动终端

2、切换到被测项目路径下
```js
    cd /Users/XXX/Documents/lesFour/
```
3、安装 calabash-cucumber gem包
```js
    gem install calabash-cucumber
```
4、生成 features文件夹
```js
    calabash-ios gen
```

# 三、 集成项目
1、设置 xcode 项目  
I.复制项目target文件  
右键如下图红色部分  

{% asset_img targetcp.jpg %}


弹出提示框，选择Duplicate，弹出如窗口，选择如下图所示  

{% asset_img targetcf.jpg %}


复制出来如下图所示，名称为lesThree copy， 双击该项目，改名为lesThree-cal（根据你自己的项目名称来设置）

{% asset_img targetfm.jpg %}


II.修改复制项目的各处名称  
如下图点击，再下拉框中选择管理项目

{% asset_img targetrn.jpg %}


弹出如下窗口，修改为如图所示名称（根据你自己的项目而定）

{% asset_img targetrnfs.jpg %}


点击完成，进入如下图所示，修改为如图所示名称

{% asset_img targetrnfs1.jpg %}


III.导入calabash.framework框架  
将你项目目录下的calabash.framework拖到xcode项目中的Frameworks文件夹中，（如没有该文件夹，请创建），如图所示

{% asset_img importfm.jpg %}


再弹出的窗口中设置如图所示

{% asset_img importfmcp.jpg %}


IV.导入CFNetwork.framework    
如图所示导入

{% asset_img linkfm.jpg %}


V.设置other linker flag   
如下图所示设置

{% asset_img linkflag.jpg %}


# 四、 测试安装  
再模拟器中运行该cal项目，窗口控制台输出如下信息则表示配置成功
```js
    2015-03-06 17:27:50.105 lesThree-cal[3279:55236] Started LPHTTP server on port 37265
    2015-03-06 17:27:52.441 lesThree-cal[3279:55421] Bonjour Service Published: domain(local.) type(_http._tcp.) name(Calabash Server)
```
