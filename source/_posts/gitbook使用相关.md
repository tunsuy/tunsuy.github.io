---
title: gitbook使用相关
date: 2015-03-15 19:25:49
tags: git
categories: git
---

gitbook的安装就不多说了，网上很多，很容易

# 一、 相关问题
下面说几个自定义相关的问题

## 1、 自定义页面样式  
* 默认样式是在：`./_book/gitbook/style.css`  
ps：增加样式只会覆盖自己定义的样式，其他的不变

<!-- more -->

* 在gitbook项目根目录下，创建styles目录  
注：一定要在该目录下创建，在其他目录下创建在gitbook build的时候会被清除掉
* 在styles目录下创建`website.css`——该文件是对gitbook页面样式的自定义
* 编辑`book.json`文件：加入
```json
        "styles": {
              "website": "styles/website.css"
         },
```
* 编译`gitbook build`
* 运行`gitbook serve`——就可以看到效果了

## 2、 自定义插件  
* 增加插件"plugins": `["toggle-chapters","splitter"]`,
* 去掉插件"plugins": `["-toggle-chapters","splitter"]`,  
—在前面加上'—'即可
