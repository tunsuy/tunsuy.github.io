---
title: 基于ruby-trello的trello系统数据写入工具
date: 2015-05-15 11:19:08
tags: [ruby]
categories: ruby
keywords: [ruby,trello,excel]
description:
---

Demo详见我的github：[https://github.com/tunsuy/TSTools/blob/master/RubyTools/trello数据上传.rb](https://github.com/tunsuy/TSTools/blob/master/RubyTools/trello数据上传.rb)

# 一、 背景
目前moa测试这边经常会用excel文档写一些场景测试案例，同时，随着对开发自测能力的加强，还会在trello系统上列一份同样的研发自测案例点，为了减少一些不必要的重复工作，编写了一个工具来将excel文档中的内容根据不同的测试类和测试点自动写入trello系统

<!-- more -->

# 二、 思路
主要用到了基于ruby的trello API

1、首先要先获取trello的key值和相关权限的token值（不同的权限，token值时不一样的）

2、通过获取到的key值和token值进行trello认证

3、然后调用相关的ruby-trello API，进行相应的读写操作

4、这里我还对不同的测试类及其对应的测试点进行了分类写入，这样方便研发根据自己负责的分模块进行自测
