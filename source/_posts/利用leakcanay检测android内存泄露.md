---
title: 利用leakcanary检测android内存泄露
date: 2015-06-10 15:21:24
tags: [android,测试]
categories: android
---

leakcanary的知识就不在这里普及了，随便Google下就知道

# 一、 项目集成
直接说在项目中怎么操作，如下：  
1、在 build.gradle 中加入引用，不同的编译使用不同的引用：
```java
     dependencies {
       debugCompile 'com.squareup.leakcanary:leakcanary-android:1.3'
       releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3'
     }
```

<!-- more -->

2、在你的项目的 Application 中：
```java
    public class ExampleApplication extends Application {

      @Override public void onCreate() {
        super.onCreate();
        LeakCanary.install(this);    //加上这一句
      }}
```
这样， 重新打包安装之后 ，在操作测试该APP的过程中，如果检测到某个 activity 有内存泄露，LeakCanary 就是自动地显示一个通知。

# 二、 具体实践
具体以我们口袋助理来介绍下：

1、在build.gradle 中加入引用

2、在MoaApplication.java中的onCreate方法中加入
```java
    LeakCanary.install(this);
```
3、故意造一个内存泄露的地方  
以创建工作汇报为例：将CreateWorkReportActivity.java中的EditText变量改为静态的(static)

4、编译打包查看效果
* 这个时候安装应用到手机，会自动安装一个Leaks应用  
注：有的手机需要重启下才能看到Leaks应用，比如我用的华为手机就是这样

5、检查是否真的如我们预期的一样：在创建工作汇报的时候存在内存泄露  
创建一个工作汇报，回到汇报列表，此时我们会发现收到了一个Leaks的通知  
如图：

{% asset_img leak-notify.jpg %}


点击该通知，进入Leaks，可以查看到详细的该内存泄露的调用信息，从而定位到该内存泄露  
如图：

{% asset_img leak-info.jpg %}
