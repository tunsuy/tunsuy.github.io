# 基于Redis缓存的单点登录SSO



## 术语

CAS系统中设计5种票据： `TGC 、 ST 、 PGT 、 PGTIOU 、 PT`。其中`TGT、ST`是`CAS1.0`协议中就有的票据，`PGT、PGTIOU、PT`是`CAS2.0`协议中有的票据。

* `Ticket-granting cookie(TGC)`：存放`用户身份认证凭证的cookie`，在浏览器和CAS Server间通讯时使用，并且只能基于`安全通道传输（Https）`，是CAS Server用来明确`用户身份的凭证`；
* `Service ticket(ST)`：服务票据，服务的惟一标识码,由CAS Server发出（Http传送），`ST`是CAS`为用户签发的访问某service的票据`。用户访问service时，service发现用户没有ST，则要求用户去CAS获取ST。用户向CAS发出获取ST的请求，如果用户的`请求中包含cookie`，则CAS会`以此cookie值为key查询缓存中有无TGT`，如果存在TGT，则用此`TGT签发一个ST`，返回给用户。用户凭借ST去访问service，service拿ST去CAS验证，验证通过后，允许用户访问资源。`一个特定的服务只能有一个惟一的ST`；
* `Proxy-Granting ticket（PGT）`：Proxy Service的代理凭据。用户通过CAS成功登录某一Proxy Service后，CAS生成一个PGT对象，缓存在CAS本地，同时将PGT的值（一个UUID字符串）回传给Proxy Service，并保存在Proxy Service里。Proxy Service拿到PGT后，就可以`为Target Service（back-end service）做代理`，为其申请PT。
* `Proxy-Granting Ticket I Owe You（PGTIOU）`: PGTIOU是CAS协议中定义的一种`附加票据`，它`增强了传输、获取PGT的安全性`。PGT的传输与获取的过程：Proxy Service调用CAS的serviceValidate接口验证ST成功后，CAS首先会访问pgtUrl指向的https url，将生成的PGT及PGTIOU传输给proxy service，proxy service会以PGTIOU为key，PGT为value，将其存储在Map中；然后CAS会生成验证ST成功的xml消息，返回给Proxy Service，xml消息中含有PGTIOU，proxy service收到Xml消息后，会从中解析出PGTIOU的值，然后以其为key，在map中找出PGT的值，赋值给代表用户信息的Assertion对象的pgtId，同时在map中将其删除。
* `Proxy Ticket (PT)`：PT是用户访问`Target Service（back-end service）`的票据。如果用户访问的是一个Web应用，则Web应用会要求浏览器提供ST，浏览器就会`用cookie去CAS获取一个ST`，然后就可以访问这个Web应用了。如果用户访问的不是一个Web应用，而是一个C/S结构的应用，因为C/S结构的应用得不到cookie，所以用户不能自己去CAS获取ST，而是通过访问proxy service的接口，凭借proxy service的PGT去获取一个PT，然后才能访问到此应用。

其它说明如下：
* `Ticket Granting ticket(TGT)`：TGT是CAS为用户签发的登录票据，拥有TGT，用户就可以证明自己在CAS成功登录过。TGT封装Cookie值以及此Cookie值对应的用户信息。用户在CAS认证成功后，CAS生成`cookie（叫TGC）`，写入浏览器，同时生成一个TGT对象，放入自己的缓存，`TGT对象的ID就是cookie的值`。当HTTP再次请求到来时，如果传过来的有CAS生成的cookie，则CAS以此cookie值为key查询缓存中有无TGT ，如果有的话，则说明用户之前登录过，如果没有，则用户需要重新登录。
* `Authentication service(AS)`：认证用服务，索取Credentials，发放TGT；
* `Ticket-granting service (TGS)`：票据授权服务，索取TGT，发放ST；
* `KDC( Key Distribution Center )`：密钥发放中心；

## CAS 基本原理
### 结构体系
从结构体系看， CAS 包括两部分： CAS Server 和 CAS Client 。

##### CAS Server
CAS Server 负责完成对用户的认证工作 , 需要独立部署 , CAS Server 会处理用户名 / 密码等凭证 (Credentials) 。

##### CAS Client
负责处理对客户端受保护资源的访问请求，需要对请求方进行身份认证时，重定向到 CAS Server 进行认证。（原则上，客户端应用不再接受任何的用户名密码等 Credentials ）。

CAS Client 与受保护的客户端应用部署在一起，以 Filter 方式保护受保护的资源。

## CAS 原理和协议
### 基础模式
SSO 访问流程主要有以下步骤：
* 访问服务：SSO客户端发送请求访问应用系统提供的服务资源。
* 定向认证：SSO客户端会重定向用户请求到SSO服务器。
* 用户认证：用户身份认证。
* 发放票据：SSO服务器会产生一个随机的Service Ticket。
* 验证票据：SSO服务器验证票据Service Ticket的合法性，验证通过后，允许客户端访问服务。
* 传输用户信息：SSO服务器验证票据通过后，传输用户认证结果信息给客户端。 

## 数据结构设计
### TGT对象(HASH类型)
属性：
```sh
{
    "expirationPolicy": "XXX",     // 二进制字符串
    "lastTimeUsed": "285335482",  // 上次使用时间
    "previousLastTimeUsed": "582852df",  // 上上次使用时间
    "creationTime": "234dfea",  // TGT创建时间
    "countOfUses": "2",   // 使用次数
    "parentTgtId": "TGT-XXX",    // 父TGT
    "expiredTime": "",   // 过期时间
    "services": "XXX",  // 跳转的url，二进制字符串
    "authentication": "XXX",  // 身份信息，二进制字符串
    "expired": "true",  // 是否已失效
    "loginURL": "{login-URL}", // 登录地址  
}
```

### User-TGT对象(SET类型)
ticket.userId:{user-id}  标识一个用户的在线会话
元素：
```sh
{
    "TGT-1",
    "TGT-2"
}
```

### TGT对象(ZSET类型)
ticket.tgts: 所有TGT，按时间排序
元素：
```sh
{
    "TGT-1",  
    "TGT-2"   
}    
```

### ST对象(HASH类型)
ticket.st:{ST-id} 
属性：
```sh
{
    "expirationPolicy": "XXX",     // 二进制字符串
    "lastTimeUsed": "285335482",  // 上次使用时间
    "previousLastTimeUsed": "582852df",  // 上上次使用时间
    "creationTime": "234dfea",  // TGT创建时间
    "countOfUses": "2",   // 使用次数
    "parentTgtId": "TGT-XXX",    // 父TGT
    "expiredTime": "",   // 过期时间
    "services": "XXX",  // 跳转的url，二进制字符串
    "fromNewLogin": "true",  // 是否新的登录生成的
    "grantedTicketAlready": "true",  // 是否已有TGT关联
}
```

## redis缓存
### 查看所有keys
```sh
30.1.3.29:26661> keys *
1) "ticket.st:ST-268-kuBYeNH1jarbho0d9yonUlnQ-sso"
2) "ticket.tgts:"
3) "ticket.tgtId_stId:6f6e776f021abd3858d288b6c150fde17523604cdfffc7e4618b8d7a95664fcd"
4) "ticket.userId:f4fe1232f30646ed84b397da39041e0f"
5) "global.users.login-records:f4fe1232f30646ed84b397da39041e0f"
6) "ticket.tgt:6f6e776f021abd3858d288b6c150fde17523604cdfffc7e4618b8d7a95664fcd"
```

如果keys提示不可用，则修改redis.conf配置文件：
```sh
rename-command FLUSHALL ""
rename-command EVAL ""
rename-command FLUSHDB ""
rename-command KEYS ""
```
去掉该keys重命名

### 查看类型
```sh
30.1.3.29:26661> type "ticket.st:ST-268-kuBYeNH1jarbho0d9yonUlnQ-sso"
hash
```

### hash类型

##### 查看hash类型的所有key
```sh
30.1.3.29:26661> hkeys "ticket.st:ST-268-kuBYeNH1jarbho0d9yonUlnQ-sso"
 1) "countOfUses"
 2) "creationTime"
 3) "lastTimeUsed"
 4) "fromNewLogin"
 5) "expiredTime"
 6) "services"
 7) "grantedTicketAlready"
 8) "expirationPolicy"
 9) "parentTgtId"
10) "previousLastTimeUsed"
```

##### 查看过期时间
```sh
30.1.3.29:26661> ttl "ticket.st:ST-268-kuBYeNH1jarbho0d9yonUlnQ-sso"
(integer) 722
30.1.3.29:26661> 
30.1.3.29:26661> pttl "ticket.st:ST-268-kuBYeNH1jarbho0d9yonUlnQ-sso"
(integer) 709418
```
`ttl`为秒，`pttl`为毫秒

还有其他一些命令：`hgetall key`，`hvals key`， `hget key field`

### 有序集合zset
##### 查看集合个数
```sh
30.1.3.29:26661> zcard "ticket.tgts:"
(integer) 1
```

##### 获取所有元素
```sh
30.1.3.29:26661> zrange "ticket.tgts:" 0 -1
1) "6f6e776f021abd3858d288b6c150fde17523604cdfffc7e4618b8d7a95664fcd"
```

##### 获取指定元素分数
```sh
30.1.3.29:26661> zscore "ticket.tgts:" 6f6e776f021abd3858d288b6c150fde17523604cdfffc7e4618b8d7a95664fcd
"1589460268224"
```

### 集合set
##### 查看元素个数
```sh
30.1.3.29:26661> scard "ticket.userId:f4fe1232f30646ed84b397da39041e0f"
(integer) 1
```

##### 查看所有元素
```sh
30.1.3.29:26661> smembers "ticket.userId:f4fe1232f30646ed84b397da39041e0f"
1) "6f6e776f021abd3858d288b6c150fde17523604cdfffc7e4618b8d7a95664fcd"
```

##### 其他命令
增加元素：`sadd key member [member ...]`
随机获取元素：`srandmember langs count`
判断元是否存在：`sismember key member`
移除元素：`srem key member [member ...]`
弹出元素：`spop key [count]`