---
title: selenium远程启动浏览器
date: 2016-04-14 19:08:23
tags: [web,selenium]
categories: selenium
---

# 一、 方案一
通过selenium-server  
## 1、 相关下载
* 下载server：http://seleniumhq.org/download/
* 下载chrome-driver：  
https://sites.google.com/a/chromium.org/chromedriver/downloads

## 2、 开启selenium-server
在浏览器主机的终端上执行命令：
```js
    java -Dwebdriver.chrome.driver="D:\chromedriver.exe" -jar selenium-server.jar
```

<!-- more -->

## 3、 编写client脚本测试
接下来就可以在其他主机上编写测试脚本并执行了：以python为例
* 下载基本python的selenium库：
```js
        pip install selenium
```
* 编写示例脚本如下：
```python
        from selenium import webdriver
        from selenium.webdriver.common.desired_capabilities import DesiredCapabilities

        driver = webdriver.Remote(
           command_executor='http://200.200.105.63:4444/wd/hub',
           desired_capabilities=webdriver.DesiredCapabilities.CHROME)

        driver.get('https://200.200.169.165');
```

# 二、 方案二
直接使用driver  
## 1、 相关下载
按照1中的下载好相应的浏览器驱动

## 2、 开启chromedriver
在浏览器主机的终端上执行命令：
```js
    chromedriver.exe --whitelisted-ips="200.200.169.162"
```

## 3、 编写client脚本测试
编写示例脚本如下：
```python
    from selenium import webdriver

    driver = webdriver.Remote(
       command_executor='http://200.200.105.63:9515',
       desired_capabilities=DesiredCapabilities.CHROME)

    driver.get('https://200.200.169.165');
```
