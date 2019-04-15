python库：kombu

openstack对其进行了封装：oslo_messaging

跟openstack其他的项目一致，采用了config的方式对rabbitmq的配置进行管理。

### 配置文件
```sh
[oslo_messaging_rabbit]
rabbit_password = rabbitmq_pwd
rabbit_hosts  = 1.1.1.1,2.2.2.2
rabbit_userid = rabbit
rabbit_port = 5671
rabbit_use_ssl = true
kombu_ssl_ca_certs = /xxx/ca-cert.pem
heartbeat_timeout_threshold = 120
```
这些配置项名称都是oslo_messaging中定义好的，不能修改。

#### 加载配置文件
```python
CONF("config_path", project='test',
	 version=version.version_string())
logging.setup(CONF, "test")
```
配置文件必须在rabbitmq启动之前加载

### 服务端示例
```python
import oslo_messaging as messaging
from oslo_config import cfg
from oslo_messaging.rpc import dispatcher

CONF = cfg.CONF


def reset_rabbit_opts():
    CONF.set_override("rabbit_password", "xxxx",
                      "oslo_messaging_rabbit")
    CONF.set_override("rabbit_hosts", "1.1.1.1", "oslo_messaging_rabbit")
    CONF.set_override("rabbit_userid", "rabbit", "oslo_messaging_rabbit")
    CONF.set_override("rabbit_port", 5671, "oslo_messaging_rabbit")
    CONF.set_override("ssl", True, "oslo_messaging_rabbit")
    CONF.set_override("ssl_ca_file", "/xxx/ca-cert.pem", "oslo_messaging_rabbit")
    CONF.set_override("heartbeat_timeout_threshold", 120, "oslo_messaging_rabbit")



def start_server():
    transport = messaging.get_rpc_transport(cfg.CONF)
    target = messaging.Target(topic="test", server="host1")
    endpoints = [TestEndpoint()]
    access_policy = dispatcher.DefaultRPCAccessPolicy
    rpc_server = messaging.get_rpc_server(transport, target, endpoints,
                                          executor='eventlet', access_policy=access_policy)
    reset_rabbit_opts()
    rpc_server.start()
    rpc_server.wait()

class TestEndpoint(object):
    target = messaging.Target(version="1.0")

    def test(self):
        print("rpc server test fun.")


if __name__ == "__main__":
    start_server()
```

### 客户端示例
```python
import oslo_messaging as messaging
from oslo_config import cfg

CONF = cfg.CONF

def reset_rabbit_opts():
    CONF.set_override("rabbit_password", "xxxx",
                      "oslo_messaging_rabbit")
    CONF.set_override("rabbit_hosts", "1.1.1.1", "oslo_messaging_rabbit")
    CONF.set_override("rabbit_userid", "rabbit", "oslo_messaging_rabbit")
    CONF.set_override("rabbit_port", 5671, "oslo_messaging_rabbit")
    CONF.set_override("ssl", True, "oslo_messaging_rabbit")
    CONF.set_override("ssl_ca_file", "/xxx/ca-cert.pem", "oslo_messaging_rabbit")
    CONF.set_override("heartbeat_timeout_threshold", 120, "oslo_messaging_rabbit")


def test():
    transport = messaging.get_rpc_transport(cfg.CONF)
    target = messaging.Target(topic="test", server="host1")
    rpc_client = messaging.RPCClient(transport, target)
    reset_rabbit_opts()
    rpc_client.call(ctxt={}, method="test")



if __name__ == "__main__":
    test()
```