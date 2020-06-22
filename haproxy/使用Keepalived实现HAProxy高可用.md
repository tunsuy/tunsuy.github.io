## 使用Keepalived实现HAProxy高可用
尽管HAProxy非常稳定，但仍然无法规避操作系统故障、主机硬件故障、网络故障甚至断电带来的风险。所以必须对HAProxy实施高可用方案。

下面将介绍利用Keepalived实现的HAProxy热备方案。即两台主机上的两个HAProxy实例同时在线，其中权重较高的实例为MASTER，MASTER出现问题时，另一台实例自动接管所有流量。

### 原理
在两台HAProxy的主机上分别运行着一个Keepalived实例，这两个Keepalived争抢同一个虚IP地址，两个HAProxy也尝试去绑定这同一个虚IP地址上的端口。 显然，同时只能有一个Keepalived抢到这个虚IP，抢到了这个虚IP的Keepalived主机上的HAProxy便是当前的MASTER。 Keepalived内部维护一个权重值，权重值最高的Keepalived实例能够抢到虚IP。同时Keepalived会定期check本主机上的HAProxy状态，状态OK时权重值增加。

`haproxy的相关介绍在我其他文章中有详细的介绍，这里就只介绍下keepalived的相关信息`

### keepalived简介
keepalived是集群管理中保证集群高可用的一个服务软件，其功能类似于heartbeat，用来防止单点故障。

keepalived是以VRRP协议为实现基础的，VRRP全称`Virtual Router Redundancy Protocol`，即虚拟路由冗余协议。

虚拟路由冗余协议，可以认为是实现路由器高可用的协议，即将N台提供相同功能的路由器组成一个路由器组，这个组里面有一个master和多个backup，master上面有一个对外提供服务的vip（该路由器所在局域网内其他机器的默认路由为该vip），master会发组播，当backup收不到vrrp包时就认为master宕掉了，这时就需要根据VRRP的优先级来选举一个backup当master。这样的话就可以保证路由器的高可用了。

keepalived主要有三个模块，分别是`core、check和vrrp`。core模块为keepalived的核心，负责主进程的启动、维护以及全局配置文件的加载和解析。check负责健康检查，包括常见的各种检查方式。vrrp模块是来实现VRRP协议的。

### keepalived的配置
keepalived只有一个配置文件keepalived.conf，里面主要包括以下几个配置区域，分别是`global_defs、static_ipaddress、static_routes、vrrp_script、vrrp_instance和virtual_server`。

#### global_defs区域
主要是配置故障发生时的通知对象以及机器标识
```sh
global_defs {
    notification_email {
        ts@test.com
        ts@test1.com
        ...
    }
    notification_email_from ts@from.com
    smtp_server smtp.abc.com
    smtp_connect_timeout 30
    enable_traps
    router_id LVS_DEVEL
}
```
* notification_email 故障发生时给谁发邮件通知。
* notification_email_from 通知邮件从哪个地址发出。
* smpt_server 通知邮件的smtp地址。
* smtp_connect_timeout 连接smtp服务器的超时时间。
* enable_traps 开启SNMP陷阱（Simple Network Management Protocol）。
* router_id 节点标识，通常为hostname，但不一定非得是hostname。故障发生时，邮件通知会用到。

#### static_ipaddress和static_routes区域
static_ipaddress和static_routes区域配置的是是本节点的IP和路由信息。如果你的机器上已经配置了IP和路由，那么这两个区域可以不用配置。
其实，一般情况下你的机器都会有IP地址和路由信息的，因此没必要再在这两个区域配置。
```sh
static_ipaddress {
    10.210.214.163/24 brd 10.210.214.255 dev eth0
    ...
}
static_routes {
    10.0.0.0/8 via 10.210.214.1 dev eth0
    ...
}
```
以上分别表示启动/关闭keepalived时在本机执行的如下命令：
```sh
# /sbin/ip addr add 10.210.214.163/24 brd 10.210.214.255 dev eth0
# /sbin/ip route add 10.0.0.0/8 via 10.210.214.1 dev eth0
# /sbin/ip addr del 10.210.214.163/24 brd 10.210.214.255 dev eth0
# /sbin/ip route del 10.0.0.0/8 via 10.210.214.1 dev eth0
```
注意： 请忽略这两个区域，因为我坚信你的机器肯定已经配置了IP和路由。

#### vrrp_script区域
keepalived只能做到对网络故障和keepalived本身的监控，即当出现网络故障或者keepalived本身出现问题时，进行切换。
但是这些还不够，我们还需要监控keepalived所在服务器上的其他业务进程，比如说haproxy，keepalived+haproxy实现haproxy的负载均衡高可用，如果haproxy异常，仅仅keepalived保持正常，是无法完成系统的正常工作的，因此需要根据业务进程的运行状态决定是否需要进行主备切换。
这个时候，我们可以通过编写脚本对业务进程进行检测监控。

```
vrrp_script check_haproxy {
        script  "/etc/keepalived/bin/keepalived_check.sh"
        interval 5
        fall 3
        rise 1
}
```
用来做健康检查，以便结合其他配置进行ip漂移。
上面的fail 3表示每间隔5s检查到3次失败，则将该节点置为不可用
上面的rise 1表示每间隔5s检查到1次成功，则将该节点置为可用

##### 优先级更新策略
如果配置了权重，如下：
```sh
vrrp_script checkhaproxy
{
    script "/etc/check.sh"
    interval 3
    weight -20
}
```
keepalived会定时执行脚本并对脚本执行的结果进行分析，动态调整vrrp_instance的优先级。
* 如果脚本执行结果为0，并且weight配置的值大于0，则优先级相应的增加
* 如果脚本执行结果非0，并且weight配置的值小于0，则优先级相应的减少
* 其他情况，维持原本配置的优先级，即配置文件中priority对应的值。

这里需要注意的是：
* 1） 优先级会不断的提高或者降低
* 2） 可以编写多个检测脚本并为每个检测脚本设置不同的weight
* 3） 不管提高优先级还是降低优先级，最终优先级的范围是在[1,254]，不会出现优先级小于等于0或者优先级大于等于255的情况

这样可以做到利用脚本检测业务进程的状态，并动态调整优先级从而实现主备切换。

##### 切换策略
在Keepalived集群中，其实并没有严格意义上的主、备节点，虽然可以在Keepalived配置文件中设置“state”选项为“MASTER”状态，但是这并不意味着此节点一直就是Master角色。控制节点角色的是Keepalived配置文件中的“priority”值，但是它并不控制所有节点的角色，另一个能改变节点角色的是在vrrp_script模块中设置的“weight”值，这两个选项对应的都是一个整数值，其中“weight”值可以是个负整数，`一个节点在集群中的角色就是通过这两个值的大小决定的`。

###### 不设置weight
在vrrp_script模块中，如果不设置“weight”选项值，那么集群优先级的选择将由Keepalived配置文件中的“priority”值决定，而在需要对集群中优先级进行灵活控制时，可以通过在vrrp_script模块中设置“weight”值来实现。

###### 设置weight
vrrp_script 里的script返回值为0时认为检测成功，其它值都会当成检测失败；

weight为正时，脚本检测成功时此weight会加到priority上，检测失败时不加；
* 主失败: 主priority < 从priority + weight 时会切换。

* 主成功：主priority + weight > 从priority + weight 时，主依然为主

weight 为负时，脚本检测成功时此weight不影响priority，检测失败时priority – abs(weight)
* 主失败: 主priority – abs(weight) < 从priority 时会切换主从
* 主成功: 主priority > 从priority 主依然为主

#### vrrp_instance和vrrp_sync_group区域
`vrrp_instance`用来定义对外提供服务的VIP区域及其相关属性。

`vrrp_sync_group`用来定义`vrrp_intance`组，使得这个组内成员动作一致。举个例子来说明一下其功能：

两个vrrp_instance同属于一个vrrp_sync_group，那么其中一个vrrp_instance发生故障切换时，另一个vrrp_instance也会跟着切换（即使这个instance没有发生故障）。
```sh
vrrp_sync_group VG1 {
    group {
        VI_1
    }
    notify_backup "/etc/keepalived/bin/notify_backup.sh"
    notify_master "/etc/keepalived/bin/notify_master.sh"
    notify_fault "/etc/keepalived/bin/notify_fault.sh"
}

vrrp_instance VI_1 {
    state BACKUP
    nopreempt
    interface eth0
    virtual_router_id 254
    priority 80
    advert_int 1

    authentication {
        auth_type AH
        auth_pass K!f26b90a0743cdf9591024c5e533b7152
    }

    virtual_ipaddress {
        30.3.3.61/16
    }

    track_script {
        check_haproxy
    }
}
```
* notify_master/backup/fault 分别表示切换为主/备/出错时所执行的脚本。
* state 可以是MASTER或BACKUP，不过当其他节点keepalived启动时会将priority比较大的节点选举为MASTER，因此该项其实没有实质用途。
* interface 节点固有IP（非VIP）的网卡，用来发VRRP包。
* virtual_router_id 取值在0-255之间，用来区分多个instance的VRRP组播。
  注意： 同一网段中virtual_router_id的值不能重复，否则会出错。
* priority 用来选举master的，要成为master，那么这个选项的值最好高于其他机器50个点，该项取值范围是1-255（在此范围之外会被识别成默认值100）。
* advert_int 发VRRP包的时间间隔，即多久进行一次master选举（可以认为是健康查检时间间隔）。
* authentication 认证区域，认证类型有PASS和HA（IPSEC），推荐使用PASS（密码只识别前8位）。
* virtual_ipaddress ，不解释了。
* nopreempt 允许一个priority比较低的节点作为master，即使有priority更高的节点启动。
  首先`nopreemt必须在state为BACKUP的节点`上才生效（因为是BACKUP节点决定是否来成为MASTER的），其次要实现类似于关闭`auto failback`的功能需要将所有节点的state都设置为BACKUP，或者将master节点的priority设置的比BACKUP低。我个人推荐使用将所有节点的state都设置成BACKUP并且都加上nopreempt选项，这样就完成了关于autofailback功能，当想手动将某节点切换为MASTER时只需去掉该节点的nopreempt选项并且将priority改的比其他节点大，然后重新加载配置文件即可（等MASTER切过来之后再将配置文件改回去再reload一下）。

### 主备切换
keepalived中优先级高的节点为MASTER。MASTER其中一个职责就是`响应VIP的arp包`，将VIP和mac地址映射关系告诉局域网内其他主机，同时，它还会以`多播`的形式（目的地址224.0.0.18）向局域网中`发送VRRP通告，告知自己的优先级`。

网络中的所有BACKUP节点`只负责处理MASTER发出的多播包`，当发现MASTER的`优先级没自己高`，或者`没收到MASTER的VRRP通告`时，BACKUP将自己`切换到MASTER状态`，然后做MASTER该做的事：

* 1.响应arp包，

* 2.发送VRRP通告。

另外，当网络中不支持多播(例如某些云环境)，或者出现网络分区的情况，keepalived BACKUP节点收不到MASTER的VRRP通告，就会出现`脑裂(split brain)现象`，此时集群中会存在多个MASTER节点。

###### 有如下几种方式进行主备切换
1、利用keepalive自身的能力，实现ip漂移
2、增加script检测脚本：

* 探测节点浮动ip是否联通
* 检查haproxy进程是否存在
* 业务性检查

3、为了避免脑裂，可以通过在`notice_backup/fault/master`中增加回调脚本：

- 主动删除或者增加浮动ip