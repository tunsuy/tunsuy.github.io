### 简介
Keepalived监控服务器集群上的HAproxy，利用Keepalived的VIP漂移技术，若集群上的HAproxy都工作正常，则VIP与优先级别高的服务器（主服务器）绑定，当主服务器当掉时，则与从服务器绑定，而VIP则是暴露给外部访问的ip；HAproxy利用Keepalived生产的VIP对多台集群进行负载，当某台服务器当掉，则将其移除，恢复后加入集群。
#### 1、	配置文件
keepalived.conf
配置文件解析：
#####  I、global_defs区域
主要是配置故障发生时的通知对象以及机器标识
##### II、vrrp_script区域
用来做健康检查的，当检查失败时会将vrrp_instance的priority减少相应的值。
##### III、vrrp_instance和vrrp_sync_group区域
vrrp_instance用来定义对外提供服务的VIP区域及其相关属性。
vrrp_rsync_group用来定义vrrp_intance组，使得这个组内成员动作一致。
#### 2、	协议
Keepalive使用的是VRRP协议，VRRP全称 Virtual Router Redundancy Protocol，即 虚拟路由冗余协议。可以认为它是实现路由器高可用的容错协议，即将N台提供相同功能的路由器组成一个路由器组(Router Group)，这个组里面有一个master和多个backup，但在外界看来就像一台一样，构成虚拟路由器，拥有一个虚拟IP（vip，也就是路由器所在局域网内其他机器的默认路由），占有这个IP的master实际负责ARP相应和转发IP数据包，组中的其它路由器作为备份的角色处于待命状态。master会发组播消息，当backup在超时时间内收不到vrrp包时就认为master宕掉了，这时就需要根据VRRP的优先级来选举一个backup当master，保证路由器的高可用。
##### I、	查看服务器的VIP情况
```bash
[root@ ~]# ip a show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether fa:16:3e:80:33:0f brd ff:ff:ff:ff:ff:ff
    inet 10.7.3.202/16 brd 30.7.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 10.7.3.201/16 scope global secondary eth0:db
       valid_lft forever preferred_lft forever
    inet 10.7.3.200/16 scope global secondary eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe80:330f/64 scope link 
       valid_lft forever preferred_lft forever
```
#### 3、	主备切换
##### 问：keepalive有几种状态
Master、backup、fault
##### 问：什么时候进行健康检查
根据配置文件中设置的时间间隔执行健康检查脚本
##### 问：什么时候判断是否需要切换
I、 如果配置文件中state都是设置的backup（非抢占式），则根据健康检查脚本的执行结果（true/false）或者自身节点是否异常决定，如果失败或者异常则切换，VIP自动漂移到备节点。
II、如果配置文件中state设置的是master（抢占式），则根据健康检查脚本的执行结果（true/false）或者自身节点是否异常决定，此时还会根据优先级来判断是否切换，规则如下：
##### I.“weight”值为正数时
在vrrp_script中指定的脚本如果检测成功，那么Master节点的权值将是“weight值与”priority“值之和，如果脚本检测失败，那么Master节点的权值保持为“priority”值，因此切换策略为：
* Master节点“vrrp_script”脚本检测失败时，如果Master节点“priority”值小于Backup节点“weight值与”priority“值之和，将发生主、备切换。
* Master节点“vrrp_script”脚本检测成功时，如果Master节点“weight”值与“priority”值之和大于Backup节点“weight”值与“priority”值之和，主节点依然为主节点，不发生切换。
##### 2.“weight”值为负数时
在“vrrp_script”中指定的脚本如果检测成功，那么Master节点的权值仍为“priority”值，当脚本检测失败时，Master节点的权值将是“priority“值与“weight”值之差，因此切换策略为：
* Master节点“vrrp_script”脚本检测失败时，如果Master节点“priority”值与“weight”值之差小于Backup节点“priority”值，将发生主、备切换。
* Master节点“vrrp_script”脚本检测成功时，如果Master节点“priority”值大于Backup节点“priority”值时，主节点依然为主节点，不发生切换。

###### Weight默认值为2。

#### 4、	仲裁
利用keepalive的健康检查机制还可以跟三方仲裁结合，在健康检查脚本中根据仲裁结果来决定是否切换。