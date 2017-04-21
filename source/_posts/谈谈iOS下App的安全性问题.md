---
title: 谈谈iOS下App的安全性问题
date: 2016-08-20 09:22:44
tags: [iOS,安全,测试]
categories: 安全
keywords: [iOS,安全,测试,App]
description:
---

下面从几个方面谈谈App的安全性问题，也有一些解决方案建议。

# 一、 URL Scheme
在iOS官方说明中：“在多个应用程序注册了同一种URL Scheme 的时候，iOS 系统程序的优先级高于第三方开发程序。但是如果一种URL Scheme 的注册应用程序都是第三方开发的，那么这些程序的优先级关系是不确定的

针对第三方应用。这里至少有两种方法可以检测自己应用的URL Scheme是否被Hijack：  
1、应用本身可以发送一条URL Scheme请求给自己，如果自己可以接收到的话，说明URL Scheme没有被劫持，如果不能收到的话，就说明被劫持了，这时候可以提醒用户卸载有冲突的app。代码如下：

<!-- more -->

```objectivec
[[UIApplication sharedApplication] openURL:[NSURL URLWithString:@“alipay://test“]];
```
2、利用 `MobileCoreServices` 服务中的 `applicationsAvailableForHandlingURLScheme()` 方法可以所有注册了该URL Schemes的应用和处理顺序，随后应用就可以检测自己，或者别人的URL Scheme是否被劫持了。代码如下：
```objectivec
Class LSApplicationWorkspace_class = objc_getClass("LSApplicationWorkspace");
NSObject* workspace = [LSApplicationWorkspace_class performSelector:@selector(defaultWorkspace)];
NSLog(@"openURL: %@",[workspace performSelector:@selector(applicationsAvailableForHandlingURLScheme:)withObject:@"alipay"]);
```

编码建议：  
一个App如果对外的URL Scheme没有做异常数据检测，那么在传输一个异常的参数时，会导致应用崩溃。所以需要做一些异常数据的检测处理。


# 二、 Storage
iOS app安装之后的目录如下：  
1、Document目录：目录存储用户数据或其他应该定期备份的信息。  
2、AppName.app目录：这是应用程序的程序包目录，包括应用程序本身，在AppStore上下载的APP是需要脱壳的才能将程序用IDA反编译的。  
3、Library目录：这个目录下有两个子目录Caches和Preference，Caches目录存储应用程序专用的支持文件，保存应用程序再次启动过程中需要的信息，而Preference目录包含程序的偏好设置。

数据库文件：  
1、Sqlite db：存储app使用过程中的一些用户数据

编码建议：  
一些敏感数据不要存储在本地文件中，也不要打印在log日志中，如果需要存储，则需要加密存储。

2、Cache dbs  
存储一些通过NSURLConnection 产生的请求回复数据

编码建议：  
对于一些附件的链接，需要设置token有效时间，当然这是服务端需要处理的问题


# 三、 Cookie
iOS开发中的Cookie一般存储在 `cookie.binary` 或者 `keychain`

1、cookie.binary  
保存在 `cookie.binary` 中，很容易就被别人利用（即使不越狱）  
路径： `/private/var/mobile/Application/x-x-x/Library/Cookies/Cookies.binarycookies`

2、Keychain  
iOS系统及第三方应用都会使用Keychain来作为数据持久化存储媒介，或者应用间数据共享的渠道。  
那么怎么来查看存储的内容呢，利用keychain_dumper工具。

操作步骤极其简单：  
* 首先在手机上安装keychain_dumper工具，在cydia上安装就可以了，然后使用ssh登录手机终端
* 赋予Keychain数据库可读权限
```js
# cd /private/var/Keychains/ 
# ls 
TrustStore.sqlite3  accountStatus.plist  caissuercache.sqlite3  keychain-2.db  keychain-2.db-shm  keychain-2.db-wal  ocspcache.sqlite3  ocspcache.sqlite3-shm  ocspcache.sqlite3-wal 
# chmod +r keychain-2.db
```
* 使用Keychain-Dumper导出Keychain
```js
Primer:/private/var/Keychainsroot# /your_path/keychain_dumper > keychain-export.txt
```
然后拷贝到本地查看，到底iOS系统和第三方应用都存放了哪些信息，就一览无余了。

Cookie安全设计的几点建议：
* Cookie 存放在 非Keychain 中，那么就应该加 随机数和Token 双层加密，虽然还有被逆向的可能，但是将难度大大增加
* Cookie 存放在 非Keychain 中，加入越狱检测代码，如果检测到越狱就提醒不要存放或者登陆，例子是 支付宝的指纹功能在越狱机提示不能用。
* 将Cookie的一部分 存放在 Keychain中，并且加密，另外一部分存放在别处(或者就在 另一处Keychain)，被逆向的难度也大大增加.
* 将网页Cookie和App Cookie 的验证分开，这样的情况还没遇到过

# 四、 安全检测CheckList
附件：点击即可下载

{% asset_link iOS安全检查点.xlsx %}

