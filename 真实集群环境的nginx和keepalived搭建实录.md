# 真实集群环境的nginx和keepalived搭建实录

## 安装nginx
因为nginx只有源码包，因此需要我们自己编译安装，另外它依赖一些其他软件，如下所示：
```sh
[root@qsy22 nginx_pkg]# ll
-rw-------. 1 root root    6209464 Oct 24 12:19 cpp-4.8.5-4.h1.x86_64.rpm
-rw-------. 1 root root   16884628 Oct 24 12:19 gcc-4.8.5-4.h1.x86_64.rpm
-rw-------. 1 root root    7494240 Oct 24 12:19 gcc-c++-4.8.5-4.h1.x86_64.rpm
-rw-------. 1 root root    1099796 Oct 24 12:19 glibc-devel-2.17-111.h9.x86_64.rpm
-rw-------. 1 root root     679572 Oct 24 12:19 glibc-headers-2.17-111.h9.x86_64.rpm
-rw-------. 1 root root    3334832 Oct 24 12:19 kernel-headers-3.10.0-327.36.58.4.x86_64.rpm
-rw-------. 1 root root      51928 Oct 24 12:19 libmpc-1.0.1-3.x86_64.rpm
-rw-------. 1 root root     430404 Oct 24 12:19 make-3.82-21.x86_64.rpm
-rw-------. 1 root root     208628 Oct 24 12:19 mpfr-3.1.1-4.x86_64.rpm
-rw-------. 1 root root     490928 Oct 24 12:19 pcre-devel-8.32-15.1.x86_64.rpm
-rw-------. 1 root root      50972 Oct 24 12:19 zlib-devel-1.2.7-15.x86_64.rpm
```

安装依赖包：
```sh
[root@qsy22 nginx_pkg]# find ./ -name "*.rpm" | xargs rpm -ivh --force --nodeps
warning: ./gcc-4.8.5-4.h1.x86_64.rpm: Header V3 RSA/SHA1 Signature, key ID 381d7ac3: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:mpfr-3.1.1-4                     ################################# [  9%]
   2:libmpc-1.0.1-3                   ################################# [ 18%]
   3:cpp-4.8.5-4.h1                   ################################# [ 27%]
   4:kernel-headers-3.10.0-327.36.58.4################################# [ 36%]
   5:glibc-headers-2.17-111.h9        ################################# [ 45%]
   6:glibc-devel-2.17-111.h9          ################################# [ 55%]
   7:gcc-4.8.5-4.h1                   ################################# [ 64%]
   8:gcc-c++-4.8.5-4.h1               ################################# [ 73%]
   9:pcre-devel-8.32-15.1             ################################# [ 82%]
  10:make-1:3.82-21                   ################################# [ 91%]
  11:zlib-devel-1.2.7-15              ################################# [100%]
```

安装nginx，进入nginx解压目录中：
```sh
[root@qsy22 nginx-1.19.1]# ./configure && make && make install

```

## 配置nginx
为了搭建一个可用的真实环境，中间遇到了很多问题，经过多方调整，最终的配置文件如下所示：
```sh

user  Tomcat LEGO;
worker_processes  auto;

error_log  logs/error.log debug;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    autoindex off;
    server_tokens off;
    port_in_redirect off;

    include       mime.types;
    default_type  application/octet-stream;

    log_format access '$remote_addr - $remote_user  [$time_local] "$request" port "$server_port"   upstream_addr "$upstream_addr" upstream_response_status $status  body_bytes_sent $body_bytes_sent  http_referer "$http_referer_filter"  user_agent "$http_user_agent"  upstream_response_time $upstream_response_time  request_time $request_time';
    access_log  logs/access.log access;

    sendfile        on;

    underscores_in_headers on;
    ignore_invalid_headers  on;

    keepalive_timeout  65;

    upstream drservers {
        server 51.6.196.200:9443;
#        server 51.6.196.201:9443;
#        server 51.6.196.211:9443;
    }

    server {
        listen       51.6.196.202:8081;
        #server_name  51.6.196.202;

        ssl on;
        ssl_session_timeout 35m;
        ssl_protocols TLSv1.2;
        ssl_ciphers         EECDH+ECDSA+AESGCM:EECDH+aRSA+AESGCM:EECDH+ECDSA+SHA384:EECDH+ECDSA+SHA256:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH:EDH+aRSA:!aNULL:!eNULL:!NULL:!LOW:!3DES:!MD5:!EXP:!SRP:!DSS:!PSK:!RC4:!SEED:!ARIA:!CAMELLIA:!DHE-RSA-AES256-SHA256:!DHE-RSA-AES256-SHA:!DHE-RSA-AES128-SHA256:!DHE-RSA-AES128-SHA:!ECDHE-RSA-AES256-SHA384:!ECDHE-RSA-AES256-SHA:!ECDHE-RSA-AES128-SHA256:!ECDHE-RSA-AES128-SHA;
        ssl_certificate /usr/local/nginx/ssl/server.crt;
        ssl_certificate_key /usr/local/nginx/ssl/server.key;
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;
        ssl_session_tickets off;
		
        location / {
            proxy_pass https://drservers/;
            proxy_set_header   Host             $server_addr:$server_port;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }

    }

    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }


}
		
```

## 启动nginx
```sh
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
```

在三个节点都执行以上操作即可


## 安装keepalive
由于keepalived没有带configure，因此需要我们自己先配置，命令如下：
```sh
[root@qsy22 keepalived-2.0.20]# aclocal
[root@qsy22 keepalived-2.0.20]# autoheader 
[root@qsy22 keepalived-2.0.20]# autoconf 
[root@qsy22 keepalived-2.0.20]# automake --add-missing
```

注意：这两个命令需要如下依赖包：
```sh
-rw-------.  1 root root     718116 Oct 24 12:57 autoconf-2.69-11.noarch.rpm
-rw-------.  1 root root     262676 Oct 24 12:59 m4-1.4.16-10.x86_64.rpm
-rw-------.  1 root root      17660 Oct 24 13:17 perl-Thread-Queue-3.02-2.noarch.rpm
-rw-------.  1 root root     695828 Oct 24 13:05 automake-1.13.4-3.noarch.rpm
```

注意：在安装过程中，会遇到openssl不匹配的问题，2.0.x版本的keepalived需要配套1.1.1以上版本的openssl
由于欧拉2.8版本的才提供了1.1.1版本的openssl，因此，如果环境不是欧拉2.8的，则只能自己去rpm仓库下载：https://rpmfind.net/linux/rpm2html/search.php?query=openssl&submit=Search+...&system=&arch=
主要是这几个包：
```sh
[root@qsy22 keepalived-2.0.20]# rpm -qa | grep ssl |grep 1.1.
openssl-1.1.1c-15.el8.x86_64
openssl-libs-1.1.1c-15.el8.x86_64
openssl-devel-1.1.1c-15.el8.x86_64
```

## 配置keepalived
```sh
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL
}

vrrp_script check_nginx {
    script "/usr/local/nginx/sbin/check_nginx.sh"
    interval 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 202
    priority 100
    advert_int 1
    nopreempt

    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
       51.6.196.202/19
    }

    track_script {
        check_nginx
    }

}
```

## 启动keepalived
```sh
/usr/local/sbin/keepalived -f /usr/local/etc/keepalived/keepalived.conf
```

## 查看vip绑定情况
```sh
[root@QSY33 nginx-1.19.1]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 28:6e:d4:89:1c:84 brd ff:ff:ff:ff:ff:ff
    inet 51.6.196.201/19 brd 51.6.223.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet 51.6.196.202/19 scope global secondary eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::2a6e:d4ff:fe89:1c84/64 scope link 
       valid_lft forever preferred_lft forever
```
可以看到绑定成功了。

## 问题集锦
### 1、访问不了vip
vip是绑定成功了，该vip访问不了，ping不通。通过查询多方文档，尝试了多种方式，如下：
* 1、清空arp：
```sh
arp -n|awk '/^[1-9]/{system("arp -d "$1)}'
```
* 2、可能被防火墙挡住了，查看如下:
```sh
[root@qsy22 eReplication]# iptables -nL
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
DROP       all  --  0.0.0.0/0            51.6.196.202        
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 13

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
DROP       all  --  51.6.196.202         0.0.0.0/0           
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
DROP       icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 14
```
可以发现确实有这么一条规则，因为是测试环境，我这里就直接将整个iptables清空了：
```sh
iptables -F
```
此时再次访问该ip，是通的。

那么这一条规则是怎么加上去的呢？通过查询官方文档，得知：
* 是因为配置文件中加了这么一个配置项：`vrrp_strict`，这个配置项什么意思呢？这个选项是严格执行VRRP协议，此模式不支持节点单播，具体可以查看官方介绍。

### 2、nginx无法集群启动
nginx在没有浮动ip的节点上没法通过该浮动ip启动，会报如下错误：
```sh
[root@aaauto eReplication]# /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
nginx: [emerg] bind() to 51.6.196.202:8081 failed (99: Cannot assign requested address)
```
这时需要修改系统配置：
```sh
net.ipv4.ip_nonlocal_bind = 1
net.ipv6.ip_nonlocal_bind = 1
```
在系统配置文件`/etc/sysctl.conf`加上上面这两行，并重启：
```sh
sysctl -p
```

### 3、nginx无法配置ssl
当我们在配置文件nginx.conf文件中增加如下配置时：
```sh
ssl on;
ssl_session_timeout 35m;
ssl_protocols TLSv1.2;
...
```
再次启动服务的时候，会报不支持ssl这样的参数。原因是我们在编译nginx的时候，没有将ssl模块编译进去。
那再已经安装了nginx的情况下，怎么办呢？难道重新安装一遍，那可不用，nginx支持增量编译，具体如下：
* 1、进入nginx的包解压目录，获取重新解压也行。
* 2、执行以下命令：
```sh
./configure --with-http_ssl_module  //重新添加这个ssl模块
make // 但是不要执行make install，因为make是用来编译的，而make install是安装，不然你整个nginx会重新覆盖的
```
* 3、在我们执行完做命令后，我们可以查看到在nginx解压目录下，objs文件夹中多了一个nginx的文件，这个就是新版本的程序了。首先我们把之前的nginx先备份一下，然后把新的程序复制过去覆盖之前的即可。
```sh
cp /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.bak
cp objs/nginx /usr/local/nginx/sbin/nginx
```
* 4、检查是否安装ssl模块成功：
```sh
[root@QSY33 nginx-1.19.1]# /usr/local/nginx/sbin/nginx -V
nginx version: nginx/1.19.1
built by gcc 4.8.5 20150623 (EulerOS 4.8.5-4) (GCC) 
built with OpenSSL 1.1.1c FIPS  28 May 2019
TLS SNI support enabled
configure arguments: --with-http_ssl_module
```
可以看到ssl已经编译进入了。

### 4、nginx无法显示后端tomacat界面
当通过nginx的监听ip端口访问的时候，出现了如下几个问题：
* 1、https://51.6.196.202.8081被重定向到了https://drservers:9443，原因是nginx具有如下的重定向规则：http://fufeixiang.com/blogs/nginx%E8%AF%B7%E6%B1%82%E5%A4%B4proxy_redirect/，https://www.jianshu.com/p/3adcb8b931a3
* 2、https://51.6.196.202.8081被重定向到了https://51.6.196.202:9443
  最终解决问题的的配置如下：
```sh
port_in_redirect off;
proxy_set_header   Host             $server_addr:$server_port;
proxy_set_header   X-Real-IP        $remote_addr;
proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
```
这些内部自定义参数可以查看相关官方文档

### 5、界面登录报403
经过debug调试，最终发现，后端server收到的请求header中的session_id为空，认证不过，那么为啥会出现这样的问题呢？经过google相关文档，发现nginx会将带下划线header和不符合规范的header全部过滤掉，但是它提供了相应的参数来忽略这种检查：
```sh
underscores_in_headers on;
ignore_invalid_headers  on;
```
在配置文件中加上这两行，即可解决问题


### 6、session共享问题
计划是采用redis来做分布式session共享