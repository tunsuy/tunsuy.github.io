---
title: iOS：https/ATS改造过程
date: 2016-06-21 13:14:11
tags: [ios,https]
categories: ios
---

# 一、 背景
苹果官方宣称今年底将全面强制启动https的支持  
1、不安全的HTTP链接将会遭到拦截  

2、而且系统 Foundation 框架下的相关网络请求，将不再默认使用 HTTP等不安全的网络协议，而默认采用 TLS 1.

# 二、 服务端改造
1、服务器需要改造的地方：
* ATS要求TLS1.2或者更高，TLS 是 SSL 新的别称。
* 通讯中的加密套件配置要求支持列出的正向保密。
* 数字证书必须使用sha256或者更高级的签名哈希算法，并且保证密钥是2048位及以上的RSA密钥或者256位及以上的ECC密钥。

2、服务器ATS在线检查：   https://www.qcloud.com/product/ssl#userDefined10

<!-- more -->

# 三、 官方建议
1、苹果官方是推荐使用NSURLSession去做HTTP请求的  

2、虽然说NSURLConnection同样支持ATS方面的特性，两者的默认行为上有些不一样  
所以应该尽早切换到NSURLSession上，避免产生一些不必要错误。

# 四、 项目改造
1、要么将info.plist中的allow全部改成NO

2、要么在提交审批的时候向苹果说明

# 五、 测试
1、抓包查看所有请求情况  
ps：推荐Charles——mac下很好用的一款抓包软件

2、在项目im中浏览https网址和http网址  
https://www.baidu.com——SSL错误  
原因： 百度自身服务器ssl协议没有指定支持TSLv1.0  
解决：(参考)
* 服务器自身配置下
* 在Info.plist里配置，指定支持TSLv1.0
https://github.com/sinaweibosdk/weibo_ios_sdk

# 六、 参考链接
http://www.liuchungui.com/blog/2015/10/11/ios9zhi-gua-pei-ats/
https://my.oschina.net/vimfung/blog/494687?_t_t_t=0.1896578531749944
https://onevcat.com/2016/06/ios-10-ats/
