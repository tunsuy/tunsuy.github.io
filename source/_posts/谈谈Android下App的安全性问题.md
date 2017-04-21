---
title: 谈谈Android下App的安全性问题
date: 2016-08-25 10:24:56
tags: [android,安全,测试]
categories: 安全
keywords: [android,安全,测试,App]
description:
---

下面从Android的几大组件详细的谈谈一些注意事项和安全编码建议，以及一些测试方法

# 一、 Android-activity
## 1、 注意事项
1.1、app内使用的私有Activity不应配置 `intent-filter`，如果配置了 `intent-filter` 需设置 `exported` 属性为false。  
1.2、使用默认 `taskAffinity`  
1.3、使用默认 `launchMode`  
1.4、启动Activity时不设置 `intent` 的 `FLAG_ACTIVITY_NEW_TASK` 标签  
1.5、谨慎处理接收的intent以及其携带的信息  

<!-- more -->

1.6、签名验证内部（in-house）app  
1.7、当Activity返回数据时候需注意目标Activity是否有泄露信息的风险  
1.8、目的Activity十分明确时使用显示启动  
1.9、谨慎处理Activity返回的数据，目的Activity返回的数据有可能是恶意应用伪造的  
1.10、验证目标Activity是否恶意app，以免受到intent欺骗，可用hash签名验证  
1.11、尽可能的不发送敏感信息，应考虑到启动 `public Activity` 中 `intent` 的信息均有可能被恶意应用窃取的风险

## 2、 测试方法
2.1、查看activity：
* 反编译查看配置文件AndroidManifest.xml中activity组件（关注配置了intent-filter的及未设置export=“false”的）
* 直接用RE打开安装后的app查看配置文件
* Drozer扫描:run app.activity.info -a packagename
* 动态查看：logcat设置filter的tag为ActivityManager

2.2、启动activity：
```js
adb shell：am start -a action -n package/componet
drozer: run app.activity.start –action android.action.intent.VIEW …
```
* 自己编写app调用 `startActiviy()` 或 `startActivityForResult()`
* 浏览器 `intent scheme`远程启动: `Intent scheme URL attack`

# 二、 Android-service
## 1、 intent-filter与exported组合建议
1.1、`exported` 属性明确定义  
1.2、私有service不定义 `intent-filter` 并且设置 `exported` 为false  
1.3、公开的service设置 `exported `为true, `intent-filter` 可以定义或者不定义  
1.4、内部/合作service设置 `exported` 为true, `intent-filter` 不定义  

## 2、 注意事项
2.1、只被应用本身使用的service应设置为私有  
2.2、service接收到的数据需需谨慎处理  
2.3、内部service需使用签名级别的 `protectionLevel` 来判断是否未内部应用调用  
2.4、不应在service创建(`onCreate`方法被调用)的时候决定是否提供服务,应在 `onStartCommand/onBind/onHandleIntent` 等方法被调用的时候做判断.  
2.5、当service又返回数据的时候,因判断数据接收app是否又信息泄露的风险  
2.6、有明确的服务需调用时使用显示意图  
2.7、尽量不发送敏感信息  
2.8、合作service需对合作公司的app签名做效验

## 3、 测试方法
3.1、service不像 `broadcast receicer` 只能静态注册,通过反编译查看配置文件 `Androidmanifest.xml` 即可确定service,若有导出的service则进行下一步  
3.2、方法查看service类,重点关注 `onCreate/onStarCommand/onHandleIntent` 方法  
3.3、检索所有类中 `startService/bindService` 方法及其传递的数据  
3.4、根据业务情况编写测试poc或者直接使用adb命令测试

# 三、 Android-Content Provider
## 1、 安全建议
1.1、`minSdkVersion` 不低于9  
1.2、不向外部app提供的数据的私有 `content provider` 设置 `exported=“false”` 避免组件暴露(编译api小于17时更应注意此点)  
1.3、使用参数化查询避免注入  
1.4、内部app通过 `content provid` 交换数据设置 `protectionLevel=“signature”` 验证签名  
1.5、公开的 `content provider` 确保不存储敏感数据  
1.6、提供asset文件时注意权限保护

## 2、 测试方法
2.1、反编译查看 `AndroidManifest.xml`（drozer扫描）文件定位 `content provider` 是否导出，是否配置权限，确定authority
```bash
#!bash
drozer:
run app.provider.info -a cn.etouch.ecalendar
```
2.2、反编译查找path，关键字 `addURI、hook api` 动态监测推荐使用zjdroid  
2.3、确定authority和path后根据业务编写POC、使用drozer、使用小工具 `Content Provider Helper、adb shell`(没有对应权限会提示错误)

# 四、 Android-Broadcast
## 1、 安全建议
1.1、私有广播接收器设置 `exported='false'`，并且不配置 `intent-filter`。(私有广播接收器依然能接收到同UID的广播)
```java
<receiver android:name=“.PrivateReceiver” android:exported=“false” />
```
1.2、对接收来的广播进行验证  
1.3、内部app之间的广播使用 `protectionLevel='signature'` 验证其是否真是内部app  
1.4、返回结果时需注意接收app是否会泄露信息  
1.5、发送的广播包含敏感信息时需指定广播接收器，使用显示意图  
1.6、`sticky broadcast` 粘性广播中不应包含敏感信息  
1.7、`Ordered Broadcast` 建议设置接收权限 `receiverPermission`，避免恶意应用设置高优先级抢收此广播后并执行 `abortBroadcast()` 方法。

## 2、 测试方法
2.1、查找动态广播接收器：反编译后检索 `registerReceiver()`,
```js
dz> run app.broadcast.info -a android -i
```
2.2、查找静态广播接收器：反编译后查看配置文件查找广播接收器组件，注意exported属性  
2.3、查找发送广播内的信息检索 `sendBroadcast` 与 `sendOrderedBroadcast`，注意 `setPackage` 方法于 `receiverPermission` 变量。

# 五、 Android-webview
## 1、 webview接口
1.1、Android 4.2 （api17）已经开始采用新的接口方法，`@JavascriptInterface` 代替 `addjavascriptInterface`, 有些android 2.3不再升级，浏览器需要兼容。  
1.2、在使用 `js2java` 的 `bridge` 时候，需要对每个传入的参数进行验证，屏蔽攻击代码。  
1.3、控制相关权限或者尽可能不要使用 `js2java` 的 `Bridge`。

## 2、 webview UXSS
2.1、服务端禁止 `frame` 嵌套 `X-FRAME-OPTIONS:DENY`。  
2.2、客户端使用 `setAllowFileAccess(false)` 方法禁止 `webview` 访问本地域。  
2.3、客户端使用 `onPageStarted (WebView view, String url, Bitmap favicon)` 方法在跳转前进行跨域判断。  
2.4、客户端对 `iframe object` 标签属性进行过滤。

# 六、 Android开发的安全防护
安卓应用在开发过程中有很多安全保护的方案

1、使用混淆保护,对APK代码进行基础的防护。  
2、使用伪加密保护方式，通过java代码对APK(压缩文件)进行伪加密，其修改原理是修改连续4位字节标记为”PK0102”的后第5位字节，奇数表示不加密偶数表示加密。伪加密后的APK不但可以防止PC端对它的解压和查看也同样能防止反编译工具编译。  
3、通过标志尾添加其他数据从而防止PC工具解压反编译，这样处理后把APK看做压缩文件的PC端来说这个文件被破坏了，所以你要对其进行解压或者查看都会提示文件已损坏，用反编译工具也会提示文件已损坏，但是它却不会影响在Android系统里面的正常运行和安装而且也能兼容到所有系统  
4、验证签名信息,通过本地或网络对签名的信息进行验证  
5、网上针对app防止反编译的做法是可以使用加壳技术

# 七、 安全检测CheckList
附件：点击即可下载

{% asset_link Android安全检查点.xlsx %}

