# Karbor中的路由

## 一、服务的加载及初始化
osapi-karbor 服务启动的过程中，调用 deploy.loadpp 使用 config 方式从 **api-paste.ini** 文件来 load 名为osapi_karbor 的应用，其入口在文件的 **[composite:osapi_karbor]**部分：
```python
[composite:osapi_karbor]
use = egg:Paste#urlmap
/: apiversions
/v1: openstack_karbor_api_v1
```
配置文件中的osapi_karbor 即 deploy.loadapp方法中传入的name。  
注：配置文件中的use字段，可以通过不同的场景，动态加载不同的api，比如可以自定义一个方法，通过开关初始化urlmap并返回。

加载 openstack_karbor_api_v1，它对应的composite 是：
```python
[composite:openstack_karbor_api_v1]
use = call:karbor.api.middleware.auth:pipeline_factory
noauth = request_id catch_errors noauth access_log apiv1
keystone = request_id catch_errors cloud_trace_wrap faultwrap authtoken access_log keystonecontext apiv1

```
**karbor/api/middleware/auth.py** 中的 **pipeline_factory** 方法如下：
```python
def pipeline_factory(loader, global_conf, **local_conf):
    """A paste pipeline replica that keys off of auth_strategy."""
    pipeline = local_conf[CONF.auth_strategy]
    pipeline = pipeline.split()
    filters = [loader.get_filter(n) for n in pipeline[:-1]]
    app = loader.get_app(pipeline[-1])
    filters.reverse()
    for filter in filters:
        app = filter(app)
    return app
```
它首先会读取 **karbor.conf** 中 auth_strategy 的值。我们一般会使用 token，所以它会从 local_conf 中读取keystone 的 pipeline 。

**第一步**，loader.get_filter（filter）来使用 deploy loader 来 load 每一个 filter，它会调用每个 filter 对应的入口 factory 函数，该函数在不同的 MiddleWare 基类中定义，都是返回函数 _factory 的指针（keystonemiddleware 是 函数auth_filter的指针）。  
**第二步**，app = loader.get_app(pipeline[-1])：使用 deploy loader 来 load app apiv1，调用入口函数 kangaroo.api.router: DJRouter.factory，factory 函数会调用其__init__方法，该方法里面进行了相关url跟controller的映射配置。  
**第三步**，app = filter(app)：然后调用每个 filter 来 wrapper APIRouter。
依次调用 filter 类的 factory，传入 app 做为参数。

到这里，osapi_karbor 的 loading 就结束了。在此过程中，各个 filter middleware 以及 WSGI Application 都被加载/注册，并被初始化。WSGI Server 在被启动后，开始等待 HTTP request，然后进入HTTP request 处理过程。

## 二、APIRouter
APIRouter 是 karbor 中的核心类之一，它负责分发 HTTP Request 到其管理的某个 Resource。它继承自**oslo_service/wsgi.py**中的Router类
在处理 HTTP Request 阶段，它负责接收经过 Middleware filters 处理过的 HTTP Request，再把它转发给 RoutesMiddleware 实例
```python
class Router(object):
       … 
	self.map = mapper
        self._router = routes.middleware.RoutesMiddleware(self._dispatch,
                                                          self.map)
```
RoutesMiddlewar会使用 mapper 得到 Request 的URL 对应的 controller 和 action，并设置到 Request 的 environ 中，然后再调用self.app()对象实例。  
注：这里涉及到python中的可调用对象概念。

该self.app是从Router中传入的：
```python
def _dispatch(req):
    …
    match = req.environ['wsgiorg.routing_args'][1]
    if not match:
        return webob.exc.HTTPNotFound()
    app = match['controller']
    return app

```
该方法会从 environ 中得到controller，其实是个 Resource 的实例，然后调用其 __call_ 方法，实现消息的分发。

比如karbor中的task任务为例  
在DJRouter中，在mapper中定义了task的相关映射
```python
class DJRouter(APIRouter):
    @classmethod
    def factory(cls, global_conf, **local_conf):
        return cls(ProjectMapper())

    def __init__(self, mapper):
        client_factory.init()
        dj_task_resource = task.create_resource()
        …
        mapper.connect("dj_task",
                       "/{project_id}/providers/{provider_id}/tasks",
                       controller=dj_task_resource,
                       action='list',
                       conditions={"method": ['GET']})
```
进入**kangaroo/api/task.py**中，有如下create_resource（）方法
```python
def create_resource():
    return wsgi.Resource(TaskController())
```


