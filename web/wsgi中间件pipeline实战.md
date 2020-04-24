## wsgi中间件pipeline实战

### 1、配置文件
[composite:test]
use = call:ts.wsgi.pipeline:factory
pipeline = mid1 mid2 app

[filter:mid1]
paste.filter_factory = xxx:xxx.factory

[filter:app]
paste.filter_factory = xxx:xxx.factory

### 2、解析配置文件
```python
def factory(loader, global_conf, **local_conf):
    """Service entry method, packaging a series of middleware applications.

    :param loader:
    :param global_conf:
    :param local_conf:
    :return:
    """
    pipeline = local_conf["pipeline"].split()

    app = loader.get_app(pipeline[-1])

    filters = [loader.get_filter(filter_name) for filter_name in pipeline[:-1]]
    filters.reverse()
    for _filter in filters:
        app = _filter(app)

    return app
```

### 3、中间件实例
```python
class Mid1(base.Middleware):
    """..."""

    @webob.dec.wsgify(RequestClass=wsgi.Request)
    def __call__(self, req):
        headers = req.headers
        environ = req.environ

        # todo
		# 业务逻辑

        return req.get_response(self.application)
```
这里用到了oslo_middleware三方库

### 4、应用实例
```python
class App(object):
    """..."""

    @classmethod
    def factory(cls, global_config, **local_config):
        return cls()

    @webob.dec.wsgify(RequestClass=wsgi.Request)
    def __call__(self, req):
        
		# todo
		# 业务逻辑

        return webob.Response(body=resp.content,
                              status=resp.status_code,
                              headerlist=resp.headers.items())
```

### 5、启动服务
```python
def main():
    CONF(sys.argv[1:], project="test", prog="test")
    logging.setup(CONF, "test")

    launcher = service.get_launcher()
    server = service.Service("test")
    launcher.launch_service(server)
    launcher.wait()
```
这里用到了oslo_service三方库