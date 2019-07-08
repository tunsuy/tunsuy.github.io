## 一、TCP termination_state
haproxy的tcplog或httplog提供了一个"termination_state"字段，这个字段提供了一个session是如何中断的指示器。在tcplog中是2个字符，在httplog中是4个字符， 通常我们初步定位故障是用前两个字符。
### 1、第一个字符
该含义表示什么事件导致了session中断:
- C : TCP session 由于client原因意外终止.
- S : TCP session 由于server原因意外终止，或者被server明确拒绝.
- P : session过早的被proxy终止。原因可以是实施了connection limit，或者被匹配了一个DENY acl，或者由于server回应的信息中带有某些危险的错误，没有通过安全检查，而被阻止(例如: 可缓存的cookie)
- L : session被haproxy本地处理，没有传递给后端server。这个标志通常发生在stats和redirect操作.
- R : proxy上的某个资源已经耗尽(内存, socket, 源端口...). 通常这个标志出现在connection阶段，系统日志内也会包含一个同样的错误信息，如果出现了这样的错误，那么必须尽快处理。
- I : proxy自检出现内部错误。这个错误应该永远不会出现，如果你在log中发现这个错误，那么通常是haproxy出现了bug。通常的解决方法是冷重启haproxy。
- D : session被haproxy关闭，因为后端server被检测到宕机。
- U : session被haproxy关闭，当后端server failback的时候关闭backup server的连接.
- K : session被haproxy的管理操作主动关闭.
- c : client-side timeout，等待client收发数据超时.
- s : server-side timeout，等待server手法数据超时.
- - : session正常结束.

### 2、第二个字符
该字符表示当session被关闭时，该session当时所处的状态:
- R : proxy正等待连接完成，等待client的合法的REQUEST. 此时尚未向任何server发送过数据.(仅用于HTTP mode)
- H : proxy正等待连接完成，等待server的合法respone HEADERS.(仅用于HTTP mode)
- Q : proxy正在队列中等待进行连接。这个标志当server设置了'maxconn'参数会出现.当持续试图redispatch请求到一个垂死的server的时候，也有可能出现在全局queue中。 如果没有redispatch发生，则不会与任何server试图建立连接.
- C : proxy正等待与server建立连接.
- D : session处于DATA阶段.
- L : proxy正在向client传送最后的data，此时server已经完成传送。这个状态很罕见，只会发生在client在接收最后的数据包时突然死掉。
- T : requset被故意延迟了，连接在整个"timeout tarpit"期间一直保持打开或者直到client关闭，两者都会被计入"Tw"计时器项目中.
- - : session正常结束.

## 二、haproxy中的log
haproxy的log记录了每个request的相关信息，典型的HTTP log格式如下:
```sh
2019-07-06 15:48:12.503 localhost haproxy [pid:29670]info  30.8.3.100:49858 [06/Jul/2019:15:48:10.953] karbor_8799~ b_def_karbor_8799/controller1 34/0/95/1420/1550 200 473 - - ---- 5/5/5/5/0 0/0 "GET /v1/9b47d36a3ee44b42b45caf70b90f82f4/protection_capabilities HTTP/1.1"
```

下面对每个字段进行相关的说明:

- `30.8.3.100:49858` 是client的IP地址和端口。根据配置的不同，这个IP可能是上级proxy的IP或是client的真实IP。
- `[06/Jul/2019:15:48:10.953]` 是TCP连接被打开的时间，之后的其他timer是基于这个时间的。
- `karbor_8799~` HTTP request请求的frontend名称，也就是配置文件中定义的名称。
- `b_def_karbor_8799/controller1` 处理请求的backend(b_def_karbor_8799)名称和真正处理请求的server(controller1)，也就是在配置文件中定义的。
- `34/0/95/1420/1550` 几个timer，这是查故障的重点关注对象，下一节详细说明。
- `200` HTTP响应状态码
- `473` 发送给client的Bytes数，包括headers。
- `5/5/5/5/0` 在记录log这一时刻的队列状态，后面详细说明。
- `0/0` 这个表示request何时进入的backend/server队列，排在这个request之前有多少个请求需要被处理。

## 三、问题排查
### 1、haproxy是否收到请求
日志如下：
```sh
Nov 26 07:08:16 localhost haproxy[20695]: 127.0.0.1:39508 [26/Nov/2015:07:08:16.154] http http/ -1/-1/-1/-1/410 400 187 - - PR-- 0/0/0/0/0 0/0 ""
```
说明request很可能没有被haproxy处理，可能问题出在haproxy的frontend配置或者是client到haproxy的网络有问题。

`(-1/-1/-1/-1/410)`这几个timer的数值揭示了可能的原因。数值"-1"说明了request没有到达相应的阶段就被终止了。
第一个"-1"说明HTTP request header还没有被完全接收完毕之前，连接就断掉了。
另外""可以看到这个连接没有被任何backend服务器处理。

### 2、网络是否有问题
日志如下：
```sh
Nov 26 07:21:52 localhost haproxy[20695]: 127.0.0.1:41150 [26/Nov/2015:07:21:45.446] http app/app13 7476/0/1/4/7481 302 638 - - ---- 0/0/0/0/0 0/0 "GET / HTTP/1.1"
```

注意7476这个数值，即7476ms，超过了7s的时间。这表明上层proxy和haproxy之间网络连接可能有问题。
通常来说HTTP头非常小，正常情况下一个TCP包就可以容纳，接收一个HTTP header不可能需要这么久。

## 四、haproxy内置的timer
例如：`10/0/30/69/109`
这些数字就是haproxy内部的timer,单位是ms，以下是依次详细说明:
- Tq (10): 接收HTTP请求用了多长时间，不包含接收POST或PUT中数据的时间。
- Tw (0): HTTP请求在队列中等待了多长时间，这个数值是backend队列(‘static’)和server(‘srv1’)队列中等待时间之和。
- Tc (30): 在TCP/IP层面，HAProxy连接backend server用了多长时间。
- Tr (69): backend server发送HTTP response header用了多长时间，不包含发送response data的时间。
- Tt (109): 从接受TCP连接开始，到TCP连接关闭，HTTP请求总共用了多长时间。
  高Tr值意味着应用出了问题

当log中的Tr的值很高的时候，通常意味着问题出在了server这一边。
为了进一步排查问题，在haproxy上已经不行了，需要到server服务器上去查找原因。
如果server响应非常慢，那么可能你会看到队列计数器的值也跟着增加了。

## 五、haproxy内置的队列
例如：`1/1/1/1/0`
这个数值是当时队列状态的快照，以下是依次详细说明:
- actconn: haproxy进程发送和接收的connection
- feconn: 发送到haproxy frontend的connection
- beconn: 被backend处理的connection，包括正在被处理的和在队列中等待的connection
- srv_conn: 在backend server上正在被处理的connection
- retries: 重试次数，这个数值应该为0，当连接建立以后，如果backend server挂了，则请求会get re-dispatched到其他backend server。

## 六、打印headers
有时候我们可以需要通过在haproxy的日志中增加header的打印，来定位问题，具体操作如下：
在frontend的配置中增加：
frontend http-in
    ...
    capture request header Host len 20
    capture request header Referer len 60
	...
比如我在http的request的headers中增加了时间戳，在haproxy中则可以通过设置获取到
```sh
7d36a3ee44b42b45caf70b90f82f4/quotas HTTP/1.1"
2019-07-02 20:40:56.374 localhost haproxy [pid:113]info  30.8.3.100:52808 [02/Jul/2019:20:40:56.282] karbor_8799~ b_def_karbor_8799/controller1 7/0/48/35/90 200 269 - - ---- 2/2/2/2/0 0/0 {1562092915762} "GET /v1/9b47d36a3ee44b42b45caf70b90f82f4/checkpoint_items/count?resource_type=OS::Nova::Server&resource_type=OS::Ironic::BareMetalServer HTTP/1.1"
2019-07-02 20:40:56.390 localhost haproxy [pid:113]info  30.8.3.100:52806 [02/Jul/2019:20:40:56.273] karbor_8799~ b_def_karbor_8799/controller1 10/0/40/64/115 200 9467 - - ---- 1/1/1/1/0 0/0 {1562092915760} "GET /v1/9b47d36a3ee44b42b45caf70b90f82f4/checkpoint_items?resource_type=OS::Nova::Server&resource_type=OS::Ironic::BareMetalServer&limit=10 HTTP/1.1"
```