---
title: linux下部署redmine
date: 2015-03-04 19:23:51
tags: [ruby,redmine,linux]
categories: redmine
---

# 一、 安装步骤
1、安装ruby环境——使用rvm比较方便

2、使用bitnami-redmine一键式安装包  

3、远程访问不了  
——关闭防火墙即可  

<!-- more -->

4、phpmyadmin无法远程访问  
——修改配置文件：`/opt/redmine-2.6.0-3/apps/phpmyadmin/conf/httpd-app.conf`  
修改如下：
```xml
    <Directory "/opt/redmine-2.6.0-3/apps/phpmyadmin/htdocs">


    # AuthType Basic
    # AuthName phpMyAdmin
    # AuthUserFile "/opt/redmine-2.6.0-3/apache2/users"
    # Require valid-user
    AllowOverride None
        <IfModule php5_module>
                php_value upload_max_filesize 80M
    php_value post_max_size 80M
        </IfModule>

    <IfVersion < 2.3 >
    Order allow,deny
    Allow from all
    Satisfy all
    </IfVersion>
    <IfVersion >= 2.3>
    Require all granted
    # Require local
    </IfVersion>
    ErrorDocument 403 "For security reasons, this URL is only accessible using localhost (127.0.0.1) as the hostname."


    </Directory>
```

重启服务即可：
```js
/opt/redmine-2.6.0-3/ctlscript.sh restart
```

# 二、 操作redmine的数据库  
1、 redmine使用的数据库为mysql，如果在终端直接输入mysql命令，则是直接调用的你以前装的mysql，而不是redmine的mysql

2、redmine的mysql相关命令的路径为/opt/redmine-2.6.0-3/mysql/bin/

3、redmine数据库默认有几个用户
* 一是你安装时填写的用户名和密码，使用这个用户登录mysql是看不到redmine的数据库的；
* 二是root用户，密码跟你安装时填写的密码一样，这个可以看到所有的数据库
