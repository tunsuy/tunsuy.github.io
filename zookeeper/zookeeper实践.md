# zookeeper实践

## fanout
Fanout Exchange 将消息路由到绑定到它的所有队列，并且忽略路由键(Routing Key) 。如果N个队列绑定到Fanout Exchange，则当向该交换机发布新消息时，将向所有N个队列传递消息的副本。
Fanout Exchange 是广播消息路由的理想选择。

不处理路由键，只需要简单的将队列绑定到交换机上，发送到该交换机的消息都会被转发到于该交换机绑定的所有队列上，fanout 交换机由于不需要进行routingkey 的对比，直接发送所有绑定的 queue，所以转发消息是最快的。

### 1、单个queue的情况
```python
# coding=utf-8
import sys

import pika

if __name__ == '__main__':
    user_passwd = sys.argv[1]
    target_host = sys.argv[2]
    msg = sys.argv[3]
    user_name = "rabbit"
    port = 5671
    vhost = '/'
    cred = pika.PlainCredentials(user_name, user_passwd)
    conn_params = pika.ConnectionParameters(target_host,
                                            port=port, ssl=True,
                                            virtual_host=vhost,
                                            credentials=cred)
    conn_broker = pika.BlockingConnection(conn_params)
    channel = conn_broker.channel()

    channel.exchange_declare(exchange='hello-exch',
                             exchange_type='fanout',
                             passive=False,
                             durable=True,
                             auto_delete=False)
    # 创建队列 "hello-queue"
    channel.queue_declare(queue='hello-queue',
                          durable=True,
                          auto_delete=False)
    # 将队列绑定到hello-exch交换机上
    channel.queue_bind(queue='hello-queue',
                       exchange='hello-exch',
                       routing_key='hala')
    msg_props = pika.BasicProperties(delivery_mode=2)
    msg_props.content_type = 'text/plain'
    msg_props.delivery_mode = 2

    channel.basic_publish(body=msg,
                          exchange='hello-exch',
                          properties=msg_props,
                          routing_key='hala')

    channel.close()
    conn_broker.close()
    sys.exit(0)

```
运行该脚本，然后执行rabbitmqctl list_queue命令
```sh
> rabbitmqctl list_queues
warning: the VM is running with native name encoding of latin1 which may cause Elixir to malfunction as it expects utf8. Please ensure your locale is set to UTF-8 (which can be verified by running "locale" in your shell)
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
name	messages
hello-queue	1
```
我们发现确实有一个叫做hello-queue的队列上有一个消息

### 2、多个queue的情况
```python
# coding=utf-8
import sys

import pika

if __name__ == '__main__':
    user_passwd = sys.argv[1]
    target_host = sys.argv[2]
    msg = sys.argv[3]
    user_name = "rabbit"
    port = 5671
    vhost = '/'
    cred = pika.PlainCredentials(user_name, user_passwd)
    conn_params = pika.ConnectionParameters(target_host,
                                            port=port, ssl=True,
                                            virtual_host=vhost,
                                            credentials=cred)
    conn_broker = pika.BlockingConnection(conn_params)
    channel = conn_broker.channel()

    channel.exchange_declare(exchange='hello-exch',
                             exchange_type='fanout',
                             passive=False,
                             durable=True,
                             auto_delete=False)
    # 创建队列1 "hello-queue1"
	channel.queue_declare(queue='hello-queue1',
						  durable=True,
						  auto_delete=False)				  
	# 将队列1绑定到hello-exch交换机上
	channel.queue_bind(queue='hello-queue1',
					   exchange='hello-exch',
					   routing_key='queue1')
					   
	# 创建队列2 "hello-queue2"					  
	channel.queue_declare(queue='hello-queue2',
						  durable=True,
						  auto_delete=False)
	# 将队列2绑定到hello-exch交换机上
	channel.queue_bind(queue='hello-queue2',
					   exchange='hello-exch',
					   routing_key='queue2')
					   
    msg_props = pika.BasicProperties(delivery_mode=2)
    msg_props.content_type = 'text/plain'
    msg_props.delivery_mode = 2

    channel.basic_publish(body=msg,
						  exchange='hello-exch',
						  properties=msg_props,
						  routing_key='any')

    channel.close()
    conn_broker.close()
    sys.exit(0)
```
这里有几个点需要注意：
1、hello-queue1和hello-queue2的路由key是不一样的
2、发送消息时的路由key也是不一样的

运行该脚本，然后执行rabbitmqctl list_queue命令
```sh
> rabbitmqctl list_queues
warning: the VM is running with native name encoding of latin1 which may cause Elixir to malfunction as it expects utf8. Please ensure your locale is set to UTF-8 (which can be verified by running "locale" in your shell)
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
name	messages
hello-queue2	1
hello-queue1	1
```
我们发现队列hello-queue1和hello-queue2上确实都有一个消息，这就说明了fanout模式确实跟路由key无关

## direct
Direct Exchange 是 RabbitMQ 默认的 Exchange，完全根据 RoutingKey 来路由消息。
设置 Exchange 和 Queue 的 Binding 时需指定 RoutingKey（一般为 Queue Name），发消息时也指定一样的 RoutingKey，消息就会被路由到对应的Queue。
它的运作方式如下：
队列绑定到具有路由键(Routing Key) K 的交换机。
当具有路由键(Routing Key) R 的新消息到达直接交换时，如果 K = R，则将其路由到队列。
直接交换通常用于以循环方式在多个 workers（同一应用程序的实例）之间分配任务。当这样做时，消息在消费者之间而不是在队列之间是负载平衡的。

```python
# coding=utf-8
import sys

import pika

if __name__ == '__main__':
    user_passwd = sys.argv[1]
    target_host = sys.argv[2]
    msg = sys.argv[3]
    user_name = "rabbit"
    port = 5671
    vhost = '/'
    cred = pika.PlainCredentials(user_name, user_passwd)
    conn_params = pika.ConnectionParameters(target_host,
                                            port=port, ssl=True,
                                            virtual_host=vhost,
                                            credentials=cred)
    conn_broker = pika.BlockingConnection(conn_params)
    channel = conn_broker.channel()

    channel.exchange_declare(exchange='hello-exch',
                             exchange_type='direct',
                             passive=False,
                             durable=True,
                             auto_delete=False)
    # 创建队列 "hello-queue"
    channel.queue_declare(queue='hello-queue',
                          durable=True,
                          auto_delete=False)
    # 将队列绑定到hello-exch交换机上
    channel.queue_bind(queue='hello-queue',
                       exchange='hello-exch',
                       routing_key='hala')
    msg_props = pika.BasicProperties(delivery_mode=2)
    msg_props.content_type = 'text/plain'
    msg_props.delivery_mode = 2

    channel.basic_publish(body=msg,
                          exchange='hello-exch',
                          properties=msg_props,
                          routing_key='test')

    channel.close()
    conn_broker.close()
    sys.exit(0)

```
注意我们绑定的路由key是`hala`，但是我们发送消息的路由key是`test`
运行该脚本，然后执行rabbitmqctl list_queue命令：
我们发现没有该队列，说明该队列没有收到消息。

## topic
Topic Exchange 基于消息路由键(Routing Key)和用于将队列绑定到交换机的模式之间的匹配，将消息路由到一个或多个队列。
Topic Exchange 通常用于实现各种发布/订阅模式变化。Topic Exchange 通常用于消息的多播路由。

路由键中特殊匹配字符说明
*（星号）可以代替一个字。
＃（散列）可以代替零个或多个单词。

Topic Exchange的路由键条件：
必须是由英文单词列表组成，单词之间使用”.”分隔
路由键的长度最大255字节

### 1、路由key为'#'的情况
```python
# coding=utf-8
import sys

import pika

if __name__ == '__main__':
    user_passwd = sys.argv[1]
    target_host = sys.argv[2]
    msg = sys.argv[3]
    user_name = "rabbit"
    port = 5671
    vhost = '/'
    cred = pika.PlainCredentials(user_name, user_passwd)
    conn_params = pika.ConnectionParameters(target_host,
                                            port=port, ssl=True,
                                            virtual_host=vhost,
                                            credentials=cred)
    conn_broker = pika.BlockingConnection(conn_params)
    channel = conn_broker.channel()

    channel.exchange_declare(exchange='topic-exch',
                             exchange_type='topic',
                             passive=False,
                             durable=True,
                             auto_delete=False)
    # 创建队列 "topic-queue"
    channel.queue_declare(queue='topic-queue',
                          durable=True,
                          auto_delete=False)
    # 将队列绑定到topic-exch交换机上
    channel.queue_bind(queue='topic-queue',
                       exchange='topic-exch',
                       routing_key='#')
    msg_props = pika.BasicProperties(delivery_mode=2)
    msg_props.content_type = 'text/plain'
    msg_props.delivery_mode = 2

    channel.basic_publish(body=msg,
                          exchange='topic-exch',
                          properties=msg_props,
                          routing_key='q.ts')

    channel.close()
    conn_broker.close()
    sys.exit(0)

```
这里有几个点需要注意：
1、topic-queue绑定的路由key是"#"

运行该脚本，然后执行rabbitmqctl list_queue命令
```sh
> rabbitmqctl list_queues |grep topic-qu
warning: the VM is running with native name encoding of latin1 which may cause Elixir to malfunction as it expects utf8. Please ensure your locale is set to UTF-8 (which can be verified by running "locale" in your shell)
topic-queue	1
```
可以看到，虽然我们没有明确指定路由key名称，但是交换机还是将消息转发给了该队列

### 2、路由key为'*'的情况
将上面队列的绑定路由key换成'*.ts'

运行该脚本，然后执行rabbitmqctl list_queue命令
```sh
> rabbitmqctl list_queues |grep topic-qu
warning: the VM is running with native name encoding of latin1 which may cause Elixir to malfunction as it expects utf8. Please ensure your locale is set to UTF-8 (which can be verified by running "locale" in your shell)
topic-queue	2
```
可以看到，虽然我们没有明确指定路由key名称，但是交换机还是将消息转发给了该队列

最后附上一个消费者实例
## 消费者
```python
# coding=utf-8
import sys

import pika


if __name__ == '__main__':
    user_passwd = sys.argv[1]
    target_host = sys.argv[2]
    if not user_passwd:
        sys.exit(1)
    
    user_name = "rabbit"
    port = 5671
    vhost = '/'
    cred = pika.PlainCredentials(user_name, user_passwd)
    conn_params = pika.ConnectionParameters(target_host,
                                            port=port, ssl=True,
                                            virtual_host=vhost,
                                            credentials=cred)
    conn_broker = pika.BlockingConnection(conn_params)
    conn_channel = conn_broker.channel()
    # 创建一个direct类型的、持久化的、没有consumer时，队列是否自动删除exchage交换机
    conn_channel.exchange_declare(exchange='topic-exch',
                                  exchange_type='topic',
                                  passive=False,
                                  durable=True,
                                  auto_delete=False)
    # 创建一个持久化的、没有consumer时队列是否自动删除的名为“hell-queue”
    conn_channel.queue_declare(queue='topic-queue',
                               durable=True,
                               auto_delete=False)

    # 将“hello-queue”队列通过routing_key绑定到“hello-exch”交换机
    conn_channel.queue_bind(queue='topic-queue',
                            exchange='topic-exch',
                            routing_key='*.ts')

    # 定义一个消息确认函数，消费者成功处理完消息后会给队列发送一个确认信息，然后该消息会被删除
    def ack_info_handler(channel, method, header, body):
        """ack_info_handler """
        LOG.info("Consume msg successful")
        isinstance(header, dict)
        isinstance(body, dict)
        print("msg: %s" % body)
        channel.basic_ack(delivery_tag=method.delivery_tag)
        channel.basic_cancel(consumer_tag='topic-ts')
        channel.stop_consuming()

    conn_channel.basic_consume(ack_info_handler,
                               queue='topic-queue',
                               no_ack=False,
                               consumer_tag='topic-ts')
    conn_channel.start_consuming()

```