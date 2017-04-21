---
title: iOS：UIWebView与WKWebView注意事项
date: 2016-04-28 17:14:51
tags: ios
categories: ios
---

# 一、 基本介绍
1、UIWebView 使用的 JavaScriptCore 框架，交互时为 JavaScript 运行的上下文环境 JSContext 注入对象 Bridge；

2、WKWebView 使用的 WebKit 框架，交互时为 webkit.messageHandlers 注入对象

# 二、 测试注意事项
1、加载速度

2、占用内存

<!-- more -->

3、缓存问题：  
H5页面更新了，app端显示还是老的，不同的ios系统处理方式不一样

4、cookie问题  
页面很多，不断切换浏览，cookie失效等

5、跨域问题  
webkit框架不允许跨域，比如从一个http页面对https发起请求

6、request拦截问题  
有可能之前用的好好的hybrid框架，换了wkwebview之后，有些就不起作用了

7、本地html文件加载  
不同的ios系统不一样

8、手势操作

# 三、 参考链接
http://mp.weixin.qq.com/s/18xXQWboHcjybd_VtcTmUg##
