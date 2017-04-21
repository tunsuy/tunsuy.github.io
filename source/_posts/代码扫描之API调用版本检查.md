---
title: 代码扫描之API调用版本检查
date: 2016-09-18 10:09:21
tags: [ios,python,sqlite]
categories: ios
---

# 一、 项目简介
语言：python语言

该项目是对工程项目代码进行全面扫描，发现其是否有代码层面的bug  
目前第一版支持了ios下的api版本控制的扫描检查  
后续将逐渐支持其他方面的代码检查，比如是否有内存溢出，数组越界等，支持检查项可配置

完整的项目源码地址：  
[https://github.com/tunsuy/iOS-code-scan](https://github.com/tunsuy/iOS-code-scan)

<!-- more -->

该项目现已很好的服务于公司的ios项目代码扫描中，有效的检测出很多开发人员疏忽的api版本使用问题

下面只对api版本控制检查进行实现讲解

# 二、 实现思路
扫描ios项目，将所有的api方法原型依次提取出来，跟ios官方定义的版本进行比对；  
检查对那些有版本限制的api，在代码中是否有相应的条件判断

那么这就涉及到如下几个问题  
1、需要知道ios库中所有的api版本信息情况  
2、需要提取出代码中所有的api调用并重新组装成方法原型  
3、需要知道该方法原型调用是否在相应的条件判断中  
4、条件判断可能是多层嵌套的  
5、方法调用可能特别复杂，跨度可能特别大  
6、方法调用可能是多层嵌套的  

# 三、 解决方案如下
1、使用爬虫将ios库的api版本信息抓取下来，并存储在数据库中  
——主要定义三个表：framework、class、api  
2、逐个文件逐行扫描，以方法调用的固有特征（比如：[class1 fun_1.1:[class2 fun_2] fun_1.2:xx]），提取出方法原型  
——当然要完整的一个不漏的提取出所有方法，处理逻辑还是很复杂的  
3、根据方法原型读取api数据库，获取其对应的sdk版本  
4、如果有版本限制，则检查其是否处于条件判断中  
——需要保持条件判断的上下文信息，多层if嵌套的匹配等

# 四、 使用技术
爬虫：使用了python的scrapy框架  
备注：  
```sh
#创建一个爬虫项目
scrapy startproject ScrapyIOSAPI

#shell调试方法
scrapy shell "https://developer.apple.com/reference?language=objc"  

#调试举例:
response.xpath('//div[@class="task-symbols"]/div[@class="symbol clm"]/a/code/text()')

scrapy crawl spider_name—执行爬虫
```

数据库：sqlite轻量级数据库

整个项目采用Python实现：
* 数据库连接技术
* 多线程技术
* 抽象类和类继承、生成器等高级技术

# 五、 工程介绍
该工程包括两个项目

## 1、 爬虫项目
爬取所有的苹果官方object-c下的API相关信息，并保存在数据库中  
使用
* 切换到工程下的iOSAPI目录下
* 执行scrapy crawl objcApi
会在当前目录下生成IOSAPI.db数据库文件  
该数据库保存了所有API的相关信息

## 2、 扫描iOS项目
支持扫描单个文件或者项目目录，根据自己输入的参数决定  
使用
* 切换到工程下的scan_proj目录下
* 执行python main.py 需要扫描的项目路径 结果保存文件
注：路径可以是单个文件名或者项目目录

# 六、 对ios代码规范的建议
对于版本控制，统一使用宏定义  
代码不要一行写多条语句，尽量按照ios开发规范来写代码
