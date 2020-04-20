# Openstack中的rpc通信

一般称调用者为 client，被调用者为 server，client 和 server 可以分布在不同的主机中，每当 client 调用 server 时，server 端创建一个线程处理 client 端的请求，所以 server 端是一个并发服务器。  
RPC 支持同步和异步这两种调用：  
- **同步调用**：client 发送请求后保持阻塞，直到获取 server 端的返回结果后才继续往下执行。
- **异步调用**：client 发送请求后继续执行后续步骤。
 
RPC 具有简单易用的特点，但是存在两个缺点：  
- Client 需要知道 server 端的 IP/PORT 才能通信，采用 IP/PORT 标志 server 端不利于系统的维护和扩展。
- 当有大量的 client 和 server 时，client 和 server 端的关系网庞大复杂，不利于系统的维护和扩展。

AMQP 是应用层协议，它在 client 和 server 端引入了消息中间件，解耦了 client 和 server 端，支持大规模下的消息通信  
在 AMQP 的术语中，一般把 client 端称为 producer，server 端称为 consumer。除此以外，还有以下概念：  
- **Topic**: 即消息，由消息头和消息体组成，消息头部包含了 routing-key 等属性，exchange 就是根据 routing-key 把消息发送到对应的队列。
- **Exchange**: 即消息转发器，producer 先把消息发送到 exchange 中，exchange 根据消息属性把消息转发到相应的队列。
- **Queue**: 即消息队列，接收和暂存由 exchange 转发来的消息，以供 consumer 消费。
- **Bind**: 即 queue 和 exchange 之间的绑定，绑定时定义了消息的转发规则。

OpenStack 有三大常用消息中间件，**RabbitMQ，QPID 和 ZeroMQ**，它们参数和接口各异，不利于直接使用，所以 oslo.messaging 对这三种消息中间件做了抽象和封装，为上层提供统一的接口。  
**Oslo.messaging **抽象出了两类数据：  
- **Transport**: 消息中间件的基本参数，如 host, port 等信息。
- **Target**: 主要包括 exchange, topic, server (optional), fanout (defaults to False) 等消息通信时用到的参数。

OpenStack RPC 模块提供了 **rpc.call，rpc.cast, rpc.fanout_cast** 三种 RPC 调用方法，发送和接收 RPC 请求。
- **rpc.call**： 发送 RPC 请求并返回请求处理结果
- **rpc.cast**： 发送 RPC 请求无返回，与 rpc.call 不同之处在于，不需要请求处理结果的返回
- **rpc.fanout_cast**： 用于发送 RPC 广播信息无返回结果


以karbor中创建策略的调度为例  
Karbor收到一个创建策略的rest api请求之后，进入**kangaroo/api/services.py**的service类create方法中
```python
def create(self, req, **kwargs):
    …    
    try:
        plans_args = self.build_plan_body(service_args)
        created_plan = self.plans.create(req, plans_args)
        created_plan_id = created_plan['plan']['id']

        # 创建schedule_operations
        for op in operations_args:
            triggers_args = op.get('trigger')
            trigger_body = self.build_trigger_body(triggers_args)
            created_trigger = self.trigger.create(req, trigger_body)
            created_trigger_id = created_trigger['trigger_info']['id']
            created_triggers.append(created_trigger_id)

            op_body = self.build_op_body(op,
                                         created_plan_id,
                                         created_trigger_id,
                                         provider_id=provider_id)

            created_op = self.scheduled_operations.create(
                req, op_body)['scheduled_operation']
            created_op['trigger'] = created_trigger['trigger_info']
            created_ops.append(created_op)

```
在**self.scheduled_operations.create()**方法的层层调用之后，最终进入**karbor/services/operationengine/rpcapi.py**中，调用了openstack的相关api
```python
def __init__(self):
    super(OperationEngineAPI, self).__init__()
    target = messaging.Target(topic=CONF.operationengine_topic,
                              version=self.RPC_API_VERSION)
    serializer = objects_base.KarborObjectSerializer()

    client = rpc.get_client(target, version_cap=None,
                            serializer=serializer)
    self._client = client.prepare(version='1.0')

def create_scheduled_operation(self, ctxt, operation_id, trigger_id):
    return self._client.call(ctxt, 'create_scheduled_operation',
                             operation_id=operation_id,
                             trigger_id=trigger_id)
```
可以看到，topic是取的**CONF.operationengine_topic**，那么这个CONF.operationengine_topic是什么值呢，就是rpc服务端注册的时候定义的。那么karbor的rpc服务端是在哪里监听的呢？

在**service（operationengine）**的main中，定义了**binary='karbor-operationengine'**，并在初始化的时候赋值给了topic
```python
def main():
    objects.register_all()
    CONF(sys.argv[1:], project='karbor',
         version=version.version_string())
    logging.setup(CONF, "karbor")
    server = service.Service.create(binary='karbor-operationengine')
    service.serve(server)
    service.wait()

```
在service（operationengine）的start中，通过相关的topic、endpoint等相关信息，开启了rpc服务的监听
```python
target = messaging.Target(topic=self.topic, server=self.host)
endpoints = [self.manager]
endpoints.extend(self.manager.additional_endpoints)
serializer = objects_base.KarborObjectSerializer()
self.rpcserver = rpc.get_server(target, endpoints, serializer)
self.rpcserver.start()

```
其中endpoints来自于self.manager的组成。对于protection的manager定义如下
`protection_manager=kangaroo.protection.dj_manager.DJProtectionManager`

