# api消息参数的传递

在服务启动之后，WSGIServer就开始在监听client的请求了  
注：关于服务的启动，在前面几篇文档中已有详细说明，请移步查看  
那各app是如何获取到请求的相关参数的呢？比如策略的创建：  
```python
@parameter_checker.is_valid_body(dj_constants.SERVICES)
def create(self, req, **kwargs):
    """
    create a service
    :param req:
    :param kwargs:
    :return:
    """
    context = req.environ['test.context']
    services = objects.PlanList.get_all_by_project(context,
                                                   context.project_id,
                                                   None, None)
```
这就涉及到plan app的注册过程  
在**router.py**的APIRouter中，mapper中plan对应的controller是resource类型的对象
```python
class APIRouter(wsgi_common.Router):
    @classmethod
    def factory(cls, global_conf, **local_conf):
        return cls(ProjectMapper())

    def __init__(self, mapper):
        plans_resources = plans.create_resource()
```
可以看到create_resource()方法返回的是一个wsgi.py的Resource对象，  
而Resource类又继承自wsgi.py的Application类，Resource实现了Application的__call__方法，这个方法会在对象创建的时候被自动调用
```python
def __call__(self, request):
    """WSGI method that controls (de)serialization and method dispatch."""

    LOG.info(_LI("%(method)s %(url)s"),
             {"method": request.method,
              "url": request.url})

    # Identify the action, its arguments, and the requested
    # content type
    action_args = self.get_action_args(request.environ)
    action = action_args.pop('action', None)
    content_type, body = self.get_body(request)
    accept = request.best_match_content_type()

    # NOTE(Vek): Splitting the function up this way allows for
    #            auditing by external tools that wrap the existing
    #            function.  If we try to audit __call__(), we can
    #            run into troubles due to the @webob.dec.wsgify()
    #            decorator.
    return self._process_stack(request, action, action_args,
                               content_type, body, accept)
```
该方法通过解析request，得到当前请求的http字段（content_type，method类型，body等），_process_stack方法如下
```python
def _process_stack(self, request, action, action_args,
                   content_type, body, accept):
    """Implement the processing stack."""

    # Get the implementing method
    try:
        meth, extensions = self.get_method(request, action,
                                           content_type, body)
	…
# Now, deserialize the request body...
    try:
        if content_type:
            contents = self.deserialize(meth, content_type, body)
        else:
            contents = {}
	…
    # Update the action args
    action_args.update(contents)

    project_id = action_args.pop("project_id", None)
    context = request.environ.get('test.context')
    if (context and project_id and (project_id != context.project_id)):
        msg = _("Malformed request url")
        return Fault(webob.exc.HTTPBadRequest(explanation=msg))

    # Run pre-processing extensions
    response, post = self.pre_process_extensions(extensions,
       
```
meth为从控制器中根据action的值获取相应的方法（例如：**cinder.api.v1.volumes.VolumeController.create**）；   
extensions为根据控制器和action的值获取相应的扩展方法；  
```python
def pre_process_extensions(self, extensions, request, action_args):
    # List of callables for post-processing extensions
    post = []

    for ext in extensions:
        if inspect.isgeneratorfunction(ext):
            response = None

            # If it's a generator function, the part before the
            # yield is the preprocessing stage
            try:
                with ResourceExceptionHandler():
                    gen = ext(req=request, **action_args)
                    response = next(gen)
            except Fault as ex:
                response = ex

            # We had a response...
            if response:
                return response, []
```
上面的代码中，其中ext就是实际调用的方法名，可以发现**ext(req=request, \*\*action_args)**，最终是从这里将业务方法的参数传递进去的




