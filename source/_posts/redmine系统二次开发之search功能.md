---
title: redmine系统二次开发之search功能
date: 2016-07-18 12:30:24
tags: [redmine,web]
categories: redmine
---

项目地址：  
[https://github.com/tunsuy/redmine_select_search](https://github.com/tunsuy/redmine_select_search)

# 一、 项目简介
该项目主要是对redmine进行简单的二次开发，满足以下需求：  
在redmine的select选择框中，如果选项特别多，那么选择是非常考验眼力的，可能眼睛看花了都找不到  
那么就考虑对所有的select框增加搜索功能

# 二、 方案一

## 1、 下载插件
使用了chosen插件：https://harvesthq.github.io/chosen/

<!-- more -->

## 2、 添加文件到项目目录中

* 将`chosen.jquery.js`文件添加到`./apps/redmine/htdocs/public/javascripts/`

* 将`chosen.css、chosen-sprite.png`添加到 `./apps/redmine/htdocs/public/stylesheets/`  

## 3、 引入js文件

* 编辑 `./apps/redmine/htdocs/app/helpers/application_helper.rb`
在下面方法处加入：
```ruby
        def javascript_heads
            tags = javascript_include_tag('jquery-1.11.1-ui-1.11.0-ujs-3.1.1', 'application', 'chosen.jquery')
            unless User.current.pref.warn_on_leaving_unsaved == '0'
              tags << "\n".html_safe + javascript_tag("$(window).load(function(){ warnLeavingUnsaved('#{escape_javascript l(:text_warn_on_leaving_unsaved)}'); });")
            end
            tags
        end
```

## 4、 引入css文件
* 编辑 `./apps/redmine/htdocs/app/views/layouts/base.html.erb`
在<head>标签处加入：
```css
        <%= stylesheet_link_tag 'chosen', :media => 'all' %>
```

## 5、 实现
* 在`./apps/redmine/htdocs/public/javascripts/application.js`文件中添加实现代码

具体代码见项目地址

# 三、 方案二

直接Hook前端代码实现

## 1、 加入拼音支持

## 2、 实现
* `在./apps/redmine/htdocs/public/javascripts/application.js`文件中添加实现代码

具体代码见项目地址
