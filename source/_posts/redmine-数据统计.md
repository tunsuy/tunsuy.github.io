---
title: redmine-数据统计
date: 2016-10-08 20:11:06
tags: [redmine,python]
categories: redmine
---

# 一、 项目说明
语言：python语言

1、项目地址：  
[https://github.com/tunsuy/redmine_statis](https://github.com/tunsuy/redmine_statis)

2、统计维度：  
* 个人迭代bug数
* 模块迭代bug数
* 个人网上问题数
* 个人迭代工作粒度
* 个人遗留问题数

如果要加上其他维度的统计，也是非常简单的

<!-- more -->

# 二、 redmine表说明
1、表issues  
—用来存放issue的标准字段。

2、表custom_fields  
—该表字段都和创建自定义字段的web页面看到的选择项很像。

3、表custom_values  
—该表可以用custom_field_id字段和custom_fields表的id关联。 而customized_id 可以和issues表的id相关联

# 三、 表关联
1、三个表issues, custom_fields和custom_values在一起表达了这么个关系。

2、一个issue的标准字段来自issues表，扩展字段来自custom_fields表，而custom_values和前custom_fields表关联，一起表示一个issue的某个自定义字段的值。

3、当表示issue的自定义字段时，`custom_fields.type`的值是 'IssueCustomField' 而`custom_values.customized_type`的值是'Issue'.

4、所有issue的自定义字段值
可以先将custom_fields表和custom_values表关联，获得如下结果：
```sql
    select customized_id as issue_id,custom_field_id,type,name,default_value,value from custom_fields a inner join custom_values b on a.id =b.custom_field_id and a.type = 'IssueCustomField' and b.customized_type='Issue' limit 2;
```
由此可以看出redmine的设计是用记录行数来表示扩展字段的值，所以可以不受mysql表字段的限制。

# 四、 访问权限
基本知识了解：  
1、授予用户redmine_static 在指定ip下 以 密码 moatest 访问 bitnami_redmine 的 select和excute操作
```sql
    grant select,excute on bitnami_redmine.* to 'redmine_static'@'200.200.169.162' identified by 'moatest'
```
2、查询mysql所有用户
```sql
    select user,host,password from mysql.user;
```
3、刷新权限设置
```sql
    flush privileges;
```
4、查询 指定IP 下 用户redmine_static 的数据库权限
```sql
    show grants for 'redmine_static'@'200.200.169.162'\G
```
5、取消用户的操作权限
```sql
    revoke select on bitnami_redmine.* from 'redmine_static'@'200.200.169.162' identified by 'moatest';
```
6、授予所有操作权限
```sql
    grant all privileges on bitnami_redmine.* to 'redmine_static'@'200.200.169.162' identified by 'moatest';
```
7、删除用户
```sql
    drop user redmine_static@'%';
```
8、创建用户
```sql
    create user redmine_static@'%' identified by 'moatest';
```

# 五、 redmine统计说明
1、在redmine服务器中新增一个mysql用户：  
——该用户只能在169.162中以用户名和密码的方式访问  
——见如上2说明  

# 六、 使用
1、切换python环境：`pyenv activate venv2710`  
2、切换到项目路径  
3、执行：python main.py [统计开始时间 统计结束时间]
```python
    eg：python main.py 2016-9-1 2016-12-31
```
