---
title: appium-ios自动化入门
date: 2016-07-10 15:13:01
tags: [appium,ios,自动化测试]
categories: ios
---

# 一、 简介
为了更好的理解业内app自动化测试框架的原理机制，以便看是否有很好的办法在解决目前项目自动化测试存在的一些问题  

这次先介绍appium的使用过程，后续有时间会看下其源码实现机制。

在mac下配置appium的自动化测试环境在这里就不说了，网上很多

因为查了下appium针对ios的自动化测试，网上的资料比较少，且说得不是很清楚，故这篇文章主要介绍怎样使用appium自动化测试自己的项目。

这些操作步骤均是自己亲自操作并实践通过的

<!-- more -->

这里已自己实现的一个简单的ios app来一步步介绍  
这篇文章先介绍在模拟器下运行的情况

# 二、 编译ios app
命令行操作
```js
    $ cd /Users/xxx/Documents/lesFour/
    $ xcodebuild -sdk iphonesimulator
```
备注：
* 官网上的介绍中是这样写的：xcodebuild -sdk iphonesimulator6.0，表示编译成ios6版本的  
这里要说明的就是 如果你在这里指定了版本号，那么你就必须修改编译文件为对应的版本，不然运行不成功
* 这条命令会在项目目录下产生一个 build 文件夹，等下我们会用到里面的一些文件
* 关于 在命令行下编译 ios项目的知识 会在后续简单的介绍

# 三、 appium-ruby项目库
切换到你喜欢的目录下，下载appium-ruby库
```js
    $ git clone https://github.com/appium/sample-code.git

    $ cd /Users/tunsuy/sample-code/sample-code/examples/ruby/
```
因为mac自带ruby，所有这里直接更新项目依赖即可
```js
    $ gem install bundle

    $ bundle update
```

# 四、 测试项目
## 1、 检查配置是否正确
这里先运行一下官方的测试程序检查是否配置正确
* 在mac下启动一个终端，开启appium-server
```js
        $ appium
        info: Welcome to Appium v1.3.5 (REV a124a15677e26b33db16e81c4b3b34d9c6b8cac9)
        info: Appium REST http interface listener started on 0.0.0.0:4723
        info: Console LogLevel: debug
        ——启动成功
```
* 另启动一个终端
```js
        $ cd /Users/xxx/sample-code/sample-code/examples/ruby/
        $ rspec simple_test.rb
```
注：这时可以看到appium-server所在的终端正在持续打出一系列日志，然后可以看到模拟器启动并测试成功

## 2、 自动化测试自己的项目
* 拷贝项目目录下之前编译产生的 build文件夹 到 /Users/xxx/sample-code/sample-code/apps/TestApp/ 目录下，覆盖掉已有的build文件夹（你也可以先备份再覆盖）  
* 修改`/Users/xxx/sample-code/sample-code/examples/ruby/`目录下的`simple_test.rb`文件  
```js
        $ vi simple_test.rb
#修改为：

        APP_PATH = '../../apps/TestApp/build/Release-iphonesimulator/lesThree.app' 为自己的路径
```
同时将` module Calculator `整个模块注释掉，也是自己的自动化代码，你也可以先不写，先看启动效果

* 启动测试
```js
        $ rspec simple_test.rb
```
* 输出结果：
```js
        No examples found.
        Finished in 0.00012 seconds
        0 examples, 0 failures
```
因为没有写测试代码，所有这里显示 0个案例，0个错误  
程序正常被启动起来了，如下图所示：  
{% asset_img appium-ios.jpg %}
