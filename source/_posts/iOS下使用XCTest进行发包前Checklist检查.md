---
title: iOS下使用UI-Testing进行发包前Checklist检查
date: 2016-05-19 15:16:22
tags: [ios,自动化测试]
categories: ios
---

# 一、 UI Testing简介
1、UI Testing 是基于 XCTest 测试框架的。XCTest 作为 OCUnit 的替代者，目前是 iOS 单元测试框架不二之选，很多其他测试框架也基于 XCTest 封装。XCTest 有如下特点：
* 测试用例需要继承 XCTestCase
* 有类似 Junit 的 setup 或者 teardown方法
* 还算不错的 Assertions
* 和 Xcode 深度集成
* 可以使用 Xcode server 的持续集成。支持 Swift 和 Objective-C

2、那 UI Testing 在 XCTest 的基础上实际上是扩展了几个类，协议
所以本质上 UI Testing 还是 XCTest，所以写用例的时候，还是需要遵从 XCTest 的规则

<!-- more -->

3、UI Testing 需要依靠 Accessibility 来定位元素。UI Testing 可以通过你的应用提供的 Accessibility 功能来与你的应用连接，这样就解决了设备大小不一的问题。如果你重新调整了 UI 中的某些元素，你也不用重写整套测试。当然实现 Accessibility 的本质不是为了使用 UI Testing，而是为了能帮助行动不便的用户更好地使用你的应用。

# 二、 具体实践
Xcode 7中创建新工程时，可以选择是否要包含UI测试——这里就不演示了  

下面说说在已有工程的时候，怎么进行UI Testing  
1、再项目工程中增加一个target  
{% asset_img target1.jpg %}

如图中1所示点击添加UI Testing Bundle  
添加之后如图中2、3所示。

2、编写测试类或者测试方法  
在图中3所示中的.m文件中添加测试方法，  
当然实际项目中，肯定是根据各自的模块自己新建一系列的类及方法  
（注：测试类及其测试方法必须按照XCTest的命令规则及要求来）

3、运行测试用例  
可以直接在导航栏中点击test运行整个UI用例 也可以如图中所示  
{% asset_img target2.jpg %}

点击用例右边的勾运行单个用例

UI testing 也具有录制功能 如图  
{% asset_img target3.jpg %}

点击图中所示 它将重新启动app ，然后你的每一步操作都将在你鼠标所在处自动生成OC代码  
（注：由于UI testing 刚出来 所以录制出来的代码可能不是很规范 需要手动改一下。不过作为官方推出来的东西，以后肯定会变得更简单强大）

# 三、 使用建议
UI testing到底适用于什么地方呢  

个人认为用来做一些简单的发包前check list比较合适

比如 我们项目在迭代发包钱都需要测试或者开发自己跑一遍check list ，作为程序员，这些简单重复的操作交给程序来运行就好了。
因为UI testing 可以使用OC语言来写 所以对于开发来说 这将是非常简单的。

# 四、 最后
具体的UI testing API 可以参看官方文档 不多比较简单
