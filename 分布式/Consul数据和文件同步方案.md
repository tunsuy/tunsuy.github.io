# Consul数据和文件同步方案
今天这里介绍的同步方案，会使用到consul官方提供的工具：consul-template

## 什么是consul-template
Consul-Template是基于Consul的自动替换配置文件的应用。在Consul-Template没出现之前，大家大多采用的是Zookeeper、Etcd+Confd这样类似的系统。
Consul官方推出了自己的模板系统Consul-Template后，动态的配置系统可以分化为Etcd+Confd和Consul+Consul-Template两大阵营。Consul-Template的定位和Confd差不多，Confd的后端可以是Etcd或者Consul。
Consul-Template提供了一个便捷的方式从Consul中获取存储的值，Consul-Template守护进程会查询Consul实例来更新系统上指定的任何模板。当更新完成后，模板还可以选择运行一些任意的命令。

Consul-Template的使用场景
Consul-Template可以查询Consul中的服务目录、Key、Key-values等。这种强大的抽象功能和查询语言模板可以使Consul-Template特别适合动态的创建配置文件

## 下载软件
在官网下载软件之后：https://github.com/hashicorp/consul-template
将该工具上传到consul节点上，并设置可执行环境

## 模板文件
说到模板文件，那么一定需要有相应的模板语法来支撑，consul-template的模板语法是go template，Consul Template按照Go Template格式来解析模板文件。具体什么是go template，可以google相关资料。除了Go提供的模板功能外，Consul模板还提供一些其他功能，具体参考consul-template官方文档。

下面我们就简单来编写一个模板文件
```sh
[root@karbor2 djmanager]# cat /opt/consul/data_config/in.tpl 
name: {{ key "ts/test/name" }}
sex: {{ key "ts/test/sex" }}

```

## 启动服务
有了模板文件之后，我们来启动服务：
```sh
[root@karbor2 data_config]# consul-template -log-level info -template "in.tpl:/opt/consul/data_config/ts.conf"
2020/10/23 08:52:38.499731 [INFO] consul-template v0.25.1 (b11fa800)
2020/10/23 08:52:38.499758 [INFO] (runner) creating new runner (dry: false, once: false)
2020/10/23 08:52:38.500231 [INFO] (runner) creating watcher
2020/10/23 08:52:38.500347 [INFO] (runner) starting
```
参数含义如下：
* 1、log-level：日志级别
* 2、-template："in.tpl:/opt/consul/data_config/ts.conf" —— 表示将in.tpl模板文件的内容根据consul中具体的kv值转换为具体的配置文件ts.conf
  还有很多其他参数，请自行研究

查看配置文件
下面我们来看下是否有具体的文件生成：
```sh
[root@karbor2 djmanager]# cat /opt/consul/data_config/ts.conf 
name: qs
sex: body
```
可以发现，当启动该进程之后，历史的数据我们也是能够获取到并写入具体的配置文件的

## 更新配置
当配置key更新之后，我们看下是否能够实时的刷新具体的配置文件
1、更新key
```sh
[root@karbor1 consul]# ./consul kv put ts/test/name 'ts'
Success! Data written to: ts/test/name
```
2、查看日志
```sh
2020/10/23 09:26:37.272136 [DEBUG] (runner) receiving dependency kv.block(ts/test/name)
2020/10/23 09:26:37.272148 [DEBUG] (runner) initiating run
2020/10/23 09:26:37.272152 [DEBUG] (runner) checking template 24cb7125e7816f0e3d3d3a95ff5f028c
2020/10/23 09:26:37.272319 [DEBUG] (runner) rendering "in.tpl" => "/opt/consul/data_config/ts.conf"
2020/10/23 09:26:37.274418 [INFO] (runner) rendered "in.tpl" => "/opt/consul/data_config/ts.conf"
2020/10/23 09:26:37.274431 [DEBUG] (runner) diffing and updating dependencies
2020/10/23 09:26:37.274438 [DEBUG] (runner) kv.block(ts/test/name) is still needed
2020/10/23 09:26:37.274441 [DEBUG] (runner) kv.block(ts/test/sex) is still needed
2020/10/23 09:26:37.274446 [DEBUG] (runner) watching 2 dependencies
2020/10/23 09:26:37.274449 [DEBUG] (runner) all templates rendered
```
可以看到是刷新了的

3、检查具体配置文件
```sh
[root@karbor2 djmanager]# cat /opt/consul/data_config/ts.conf 
name: ts
sex: body
```
可以看到具体的配置文件也是实时的更新了的。

## 配置更新回调
一般我们都有这样一个诉求，当某个key更新之后，用到该key的业务需要能够实时的感知到，比如nginx的某个key更新之后，我们需要实时的触发刷新nginx的配置，consul-template也是支持的，下面让我们来实际操作操作。

1、更新配置：
```sh
[root@karbor1 consul]# ./consul kv put ts/test/name 'ts_1'
Success! Data written to: ts/test/name
```

2、查看日志：
```sh
2020/10/23 09:36:40.281704 [DEBUG] (runner) receiving dependency kv.block(ts/test/name)
2020/10/23 09:36:40.281716 [DEBUG] (runner) initiating run
2020/10/23 09:36:40.281720 [DEBUG] (runner) checking template 24cb7125e7816f0e3d3d3a95ff5f028c
2020/10/23 09:36:40.281875 [DEBUG] (runner) rendering "in.tpl" => "/opt/consul/data_config/ts.conf"
2020/10/23 09:36:40.285175 [INFO] (runner) rendered "in.tpl" => "/opt/consul/data_config/ts.conf"
2020/10/23 09:36:40.285200 [DEBUG] (runner) appending command "sh /home/djmanager/test_consul.sh" from "in.tpl" => "/opt/consul/data_config/ts.conf"
2020/10/23 09:36:40.285208 [DEBUG] (runner) diffing and updating dependencies
2020/10/23 09:36:40.285217 [DEBUG] (runner) kv.block(ts/test/sex) is still needed
2020/10/23 09:36:40.285220 [DEBUG] (runner) kv.block(ts/test/name) is still needed
2020/10/23 09:36:40.285224 [INFO] (runner) executing command "sh /home/djmanager/test_consul.sh" from "in.tpl" => "/opt/consul/data_config/ts.conf"
2020/10/23 09:36:40.285276 [INFO] (child) spawning: sh /home/djmanager/test_consul.sh
2020/10/23 09:36:40.287013 [DEBUG] (runner) watching 2 dependencies
2020/10/23 09:36:40.287024 [DEBUG] (runner) all templates rendered
2020/10/23 09:36:40.287056 [DEBUG] (cli) receiving signal "child exited"
```

3、检查更新：
```sh
[root@karbor2 djmanager]# cat /opt/consul/data_config/ts.conf 
name: ts_1
sex: body

```

4、检查是否执行了回调脚本
```sh
[root@karbor2 djmanager]# cat /opt/consul/data_config/test_consul.log 
receive key update, so start callback
```
可以发现我们配置的回调脚本，在配置更新的时候，被实时的调用了。