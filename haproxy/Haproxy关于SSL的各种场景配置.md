# Haproxy关于SSL的各种场景配置

此文章翻译自haproxy官方文档：https://www.haproxy.com/blog/ssl-client-certificate-management-at-application-level/


### 1、强制client提供证书
在下面的配置中，仅具有客户端证书的用户被允许在应用程序上建立连接。这可以通过关键字"verify required"来实现。
```sh
frontend ft_ssltests
 mode http
 bind 192.168.10.1:443 ssl crt ./server.pem ca-file ./ca.crt verify required
 default_backend bk_ssltests
backend ssltests
 mode http
 server s1 192.168.10.101:80 check
 server s2 192.168.10.102:80 check
```
如果客户端不提供任何证书，则HAProxy将在SSL握手期间关闭连接，测试如下：
##### 带证书
```sh
$ openssl s_client -connect 192.168.10.1:443 -cert ./client1.crt -key ./client1.key
```
##### 不带证书
```sh
$ openssl s_client -connect 192.168.10.1:443
[...]ssl handshake failure[...]
```
##### 带过期证书
```sh
$ openssl s_client -connect 192.168.10.1:443 -cert ./client_expired.crt -key ./client_expired.key 
[...]ssl handshake failure[...]
```

### 2、client选择性提供证书
在下面的配置中，所有具有证书的用户和没有证书的用户都可以连接。这可以通过关键字"verify optional"来实现。
我们可以根据是否存在证书将用户重定向到其他服务器中：
```sh
frontend ssltests
 mode http
 bind 192.168.10.1:443 ssl crt ./server.pem ca-file ./ca.crt verify optional
 use_backend sharepoint if { ssl_fc_has_crt }     # check if the certificate has been provided and give access to the application
 default_backend webmail
backend sharepoint
 mode http
 server srv1 192.168.10.101:80 check
 server srv2 192.168.10.102:80 check
backend webmail
 mode http
 server srv3 192.168.10.103:80 check
 server srv4 192.168.10.104:80 check
```
* 如果客户端不提供任何证书，则HAProxy会将其路由到Webmail。
* 如果客户端提供证书，则HAProxy会将其路由到应用程序（在我们的示例中为sharepoint）
* 如果客户端提供了过期的证书，则HAProxy会拒绝连接

### 3、忽略证书过期错误
在下面的配置中，所有具有证书的用户和没有证书的用户都可以连接。这可以通过关键字"verify optional"来实现。
选项"crt-ignore-err 10"告诉HAProxy忽略实际上与过期证书匹配的证书错误10。
我们可以根据是否存在证书将用户重定向到其他服务器，并且可以为证书已过期的用户定制一个专用页面，其中包含有关如何续订或要求新证书的过程
```sh
frontend ssltests
 mode http
 bind 192.168.10.1:443 ssl crt ./server.pem ca-file ./ca.crt verify optional crt-ignore-err 10
 use_backend static if { ssl_c_verify 10 }  # if the certificate has expired, route the user to a less sensitive server to print an help page
 use_backend sharepoint if { ssl_fc_has_crt }        # check if the certificate has been provided and give access to the application
 default_backend webmail
backend static
 mode http
 option http-server-close
 redirect location /certexpired.html if { ssl_c_verify 10 } ! { path /certexpired.html }
 server srv5 192.168.10.105:80 check
 server srv6 192.168.10.106:80 check
backend sharepoint
 mode http
 server srv1 192.168.10.101:80 check
 server srv2 192.168.10.102:80 check
backend webmail
 mode http
 server srv3 192.168.10.103:80 check
 server srv4 192.168.10.104:80 check
```
* 如果客户端不提供任何证书，则HAProxy会将其路由到Webmail。
* 如果客户端提供证书，则HAProxy会将其路由到应用程序（在我们的示例中为sharepoint）
* 如果客户端提供了过期证书，则HAProxy会将其路由到静态服务器，并强制用户显示该页面，该页面提供有关过期证书及其更新方式的说明。

### 4、忽略所有的证书错误
在下面的配置中，所有具有证书的用户和没有证书的用户都可以连接。这可以通过关键字"verify optional"来实现。
选项"crt-ignore-err all"告诉HAProxy忽略任何客户端证书错误。
选项"crl-file ./ca_crl.pem"告诉HAProxy检查在参数提供的证书吊销列表中是否尚未吊销客户端。
我们可以根据是否存在证书将用户重定向到其他服务器，并且可以为证书已过期的用户定制一个专用页面，其中包含有关如何续订或要求新证书的过程。我们还可以向其证书已被撤消的用户显示专用页面。
```sh
frontend ssltests
 mode http
 bind 192.168.10.1:443 ssl crt ./server.pem ca-file ./ca.crt verify optional crt-ignore-err all crl-file ./ca_crl.pem
 use_backend static unless { ssl_c_verify 0 }  # if there is an error with the certificate, then route the user to a less sensitive farm
 use_backend sharepoint if { ssl_fc_has_crt }           # check if the certificate has been provided and give access to the application
 default_backend webmail
backend static
 mode http
 option http-server-close
 redirect location /certexpired.html if { ssl_c_verify 10 } ! { path /certexpired.html } # SSL error 10 means "certificate expired"
 redirect location /certrevoked.html if { ssl_c_verify 23 } ! { path /certrevoked.html } # SSL error 23 means "Certificate revoked"
redirect location /othererrors.html unless { ssl_c_verify 0 } ! { path /othererrors.html }
 server srv5 192.168.10.105:80 check
 server srv6 192.168.10.106:80 check
backend sharepoint
 mode http
 server srv1 192.168.10.101:80 check
 server srv2 192.168.10.102:80 check
backend webmail
 mode http
 server srv3 192.168.10.103:80 check
 server srv4 192.168.10.104:80 check
```
* 如果客户端不提供任何证书，则HAProxy会将其路由到Webmail。
* 如果客户端提供证书，则HAProxy会将其路由到应用程序（在我们的示例中为sharepoint）
* 如果客户端提供了过期证书，则HAProxy会将其路由到静态服务器，并强制用户显示该页面，该页面提供有关过期证书及其更新方式的说明。
* 如果客户端提供了吊销的证书，则HAProxy会将其路由到静态服务器，并强制用户显示提供有关吊销证书的说明的页面（由管理员编写此页面）。
* 对于与客户端证书有关的任何其他错误，HAProxy会将用户路由到静态服务器，并强制用户显示一个页面，以说明存在错误以及如何与支持部门联系（由管理员决定）编写此页面。

### 5、根据ssl错误重定向
在下面的配置中，所有具有证书的用户和没有证书的用户都可以连接。这可以通过关键字"verify optional"来实现。
选项"crt-ignore-err all"告诉HAProxy忽略所有客户端证书。
选项"crl-file ./ca_crl.pem"告诉HAProxy检查在参数提供的证书吊销列表中是否尚未吊销客户端。
文件ca.pem包含2个CA：ca和ca2。
我们可以根据是否存在证书将用户重定向到其他服务器场，并且可以为证书已过期的用户建议一个专用页面，其中包含有关如何续订或要求新证书的过程。我们还可以向其证书已被撤消的用户显示专用页面。
```sh
frontend ssltests
 mode http
 bind 192.168.10.1:443 ssl crt ./server.pem ca-file ./ca.pem verify optional crt-ignore-err all crl-file ./ca_crl.pem
 use_backend static unless { ssl_c_verify 0 }  # if there is an error with the certificate, then route the user to a less sensitive farm
 use_backend sharepoint if { ssl_fc_has_crt }           # check if the certificate has been provided and give access to the application
 default_backend webmail
backend static
 mode http
 option http-server-close
 acl url_expired path /certexpired.html
 acl url_revoked path /certrevoked.html
 acl url_othererrors path /othererrors.html
 acl cert_expired ssl_c_verify 10
 acl cert_revoked ssl_c_verify 23
 reqadd X-Ssl-Error: 10 if cert_expired
 reqadd X-Ssl-Error: 23 if cert_revoked
 reqadd X-Ssl-Error: other if ! cert_expired ! cert_revoked
 redirect location /certexpired.html if cert_expired ! url_expired
 redirect location /certrevoked.html if cert_revoked ! url_revoked
 redirect location /othererrors.html if ! cert_expired ! cert_revoked ! url_othererrors
 server srv5 192.168.10.105:80 check
 server srv6 192.168.10.106:80 check
backend sharepoint
 mode http
 server srv1 192.168.10.101:80 check
 server srv2 192.168.10.102:80 check
backend webmail
 mode http
 server srv3 192.168.10.103:80 check
 server srv4 192.168.10.104:80 check
```
* 如果客户端不提供任何证书，则HAProxy会将其路由到Webmail。
* 如果客户端提供证书，则HAProxy会将其路由到应用程序（在我们的示例中为共享点）
* 如果客户端提供了过期证书，则HAProxy会将其路由到静态服务器（非敏感服务器），并强制用户显示该页面，该页面提供有关过期证书及其更新方式的说明（这取决于管理员来编写）这一页）。
* 如果客户端提供了吊销的证书，则HAProxy会将其路由到静态服务器（不敏感），并强制用户显示提供有关吊销证书的说明的页面（由管理员编写此页面）。
* 对于与客户端证书有关的任何其他错误，HAProxy会将用户路由到静态服务器（不敏感），并强制用户显示一个页面，以说明存在错误以及如何与支持部门联系（由管理员决定）编写此页面）。