---
title: hexo+github博客部署问题记录
date: 2015-02-20 14:46:31
tags: [blog,github,hexo]
categories: blog
---

# 一、 博客简介
现在越来越多的人开始有了自己的博客，记录一下自己的学习和生活，个人博客一般都是静态博客  

搭建博客一般有几种方式：  
1、全部自己实现：包括服务器、自己做网站、购买域名等——费时费力费钱  
2、租用云服务器，采用静态博客生成工具：但是云服务器一般也是要付费的

那有没有免费的呢？  
答案是有的，那就是github

<!-- more -->

这里再说下静态博客工具+github的原理：  
以hexo+github为例（其他的静态博客生成工具类似）  
* 静态博客一般就是帮我们将博客的外观和框架做成了模型，用户只需要专注于写文章（一般都支持使用md格式文档），然后使用博客工具命令生成静态网页；所以你将这些网页和资源放在任何web服务器上都可以运行  
* 既然放在任何的web服务器上都可以运行，那么久有了github为我们提供的免费托管服务器，我们只需要将我们的静态网页和资源上传到github上即可

hexo+github的具体安装部署这里就不多说了，网上教程很多

# 二、 问题记录
这里只记录几个常用需求问题  

## 1、 推送hexo博客内容到github
出现错误：
```js
    error: The requested URL returned error: 403 Forbidden
```
安装网上的方式都试了，都不行  
最后解决方法如下：
* 修改站点配置文件` _config.yml`
```yml
        repository: git@github.com:tunsuy/tunsuy.github.io.git
```
ps：之前是使用的https，不知道为什么不行

## 2、搜索引擎收录
因为github不允许baidu爬虫，所以提交文章链接到baidu搜索引擎比较麻烦  
参考这篇文章：  
[Hexo-优化：提交sitemap及解决百度爬虫抓取-GitHub-Pages-问题](http://www.yuan-ji.me/Hexo-优化：提交sitemap及解决百度爬虫抓取-GitHub-Pages-问题/)

## 3、字体配置
编辑 `./themes/next/source/css/_variables/custom.styl`  
推荐这样的字体族配置：  
`Helvetica, Tahoma, Arial, STXihei, "华文细黑", "Microsoft YaHei", "微软雅黑", sans-serif`

## 4、文章目录配置
编辑 `./themes/next/_config.yml`
```yml
# Table Of Contents in the Sidebar
# 控制是否显示目录
toc:
  enable: true

  # Automatically add list number to toc.
  # 控制目录序号的显示
  number: false
```
ps: 要显示多级目录时，在用md编辑文章时，目录只能依次，而不能跨级  
eg：#->## ，而不能 #->###

## 5、分类
hexo不支持一篇文章属于多个同级分类

## 6、文章内链接显示蓝色
编辑：`./themes/next/source/css/_common/components/post/post.styl`  
加入如下css样式：
```csss
.post-body p a{
  color: #0593d3;
  border-bottom: none;

  &:hover {
    color: #0477ab;
    text-decoration: underline;
  }
}
```

## 7、Next主题宽度调节
* 打开 `themes/next/source/css/_common/components/post/post-expand.styl` 文件，找到
```css
@media (max-width: 767px)
```
改为
```css
@media (max-width: 1080px)
```
* 打开 `themes/next/source/css/_variables/base.styl` 文件，找到
```css
$main-desktop                   = 960px
$main-desktop-large             = 1200px
$content-desktop                = 700px
```
修改 `$main-desktop` 和 `$content-desktop` 的数值：
```css
$main-desktop                   = 1080px
$main-desktop-large             = 1200px
$content-desktop                = 810px
```
Next.Mist 主题的文章宽度至此改完了。如果你用的是 Next.Pisces，还需要继续修改。

* 打开 `themes/next/source/css/_schemes/Pisces/_layout.styl` 文件，将第 4 行的 width改为1080px ，修改后如下：
```css
.header {
  position: relative;
  margin: 0 auto;
  width: 1080px;
 ```

## 8、 在社交栏添加QQ邮箱
先说下next添加社交链接的原理  
主要代码在 `./themes/next/layout/_macro/sidebar.swig` 文件，如下：
```html
<div class="links-of-author motion-element">
  {% if theme.social %}
	{% for name, link in theme.social %}
	  <span class="links-of-author-item">
		<a href="{{ link }}" target="_blank" title="{{ name }}">
		  {% if theme.social_icons.enable %}
			<i class="fa fa-fw fa-{{ theme.social_icons[name] | default('globe') | lower }}"></i>
		  {% endif %}
		  {{ name }}
		</a>
	  </span>
	{% endfor %}
  {% endif %}
</div>
```

下面看下怎么接入qq邮箱，主要使用qq邮箱的邮我组件  
1、在网页版qq邮箱的设置中找到邮我组件，按照提示生成html代码  
注：可以完全使用该代码生成的风格，也可以只用href='xxx'这个链接即可

2、修改next配置文件  
在 `./themes/next/_config.yml` 文件的 `social:` 处加上 `Email: xxx`。  
注：xxx就是刚才邮我组件代码中的href。  
在`social_icons:` 处加上 `Email: envelope` 这个是 `FontAwesome icon` 中的图标  
注：前面说过，也可以完全使用邮我组件自己的风格。

# 三、 好文推荐
[Hexo-SEO优化](http://www.zaoanx.com/Hexo-SEO优化.html)  
[Hexo官方文档](https://hexo.io/zh-cn/docs/)  
[主题next官方文档](http://theme-next.iissnan.com/)  
[Next优化](http://www.wuxubj.cn/2016/08/Hexo-nexT-build-personal-blog/)  
[Hexo+NexT主题配置备忘](http://blog.ynxiu.com/2016/hexo-next-theme-optimize.html)
