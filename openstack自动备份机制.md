# karbor自动执行机制

## 一、自动执行开启时机

在服务启动的时候，都会统一调用init_host方法，每个服务都需要自己实现该方法，具体见/karbor/service.py文件  
```python
    def start(self):
        ...
        ctxt = context.get_admin_context()
        try:
            service_ref = db.service_get_by_args(ctxt,
                                                 self.host,
                                                 self.binary)
            self.service_id = service_ref['id']
        except exception.NotFound:
            self._create_service_ref(ctxt)
        self.manager.init_host(service_id=self.service_id)
```
这是rpc服务初始化的时候
```python
    def start(self):
       ...

        """
        if self.manager:
            self.manager.init_host()
        self.server.start()
        self.port = self.server.port
```
这是wsgi服务初始化的时候  
然后我们分别进入各服务的manager类中查看，会发现确实都定义了init_host方法  
* karbor-api服务的init_host方法中则是启动了rabbitmq监控  
* karbor-operation服务则是 开启了自动清除item的循环检测
* karbor-protection没有做任何东西

下面就来看下karbor-operation的自动清理机制

## 二、自动清理机制
在init_host方法中，实例化了time_trigger的TriggerOperationGreenThread对象，在TriggerOperationGreenThread的init方法中调用_start方法  
```python
    def _start(self, first_run_time):
        self._running = True

        now = datetime.utcnow()
        initial_delay = 0 if first_run_time <= now else (
            int(timeutils.delta_seconds(now, first_run_time)))

        self._thread = eventlet.spawn_after(
            initial_delay, self._run, first_run_time)
        self._thread.link(self._on_done)
```
该方法调用了run方法
```python
   def _run(self, expect_run_time):
        while self._running:
            self._is_sleeping = False
            self._pre_run_time = expect_run_time

            expect_run_time = self._function(expect_run_time)
            if expect_run_time is None or not self._running:
                break

            self._is_sleeping = True

            now = datetime.utcnow()
            idle_time = 0 if expect_run_time <= now else int(
                timeutils.delta_seconds(now, expect_run_time))
            eventlet.sleep(idle_time)

```
可以看到，进入了while循环：计算出下一次执行时间，然后判断是否达到执行时间，到了则执行相应的function

那么这里的function是什么呢？那就是init_host传递进来的自动清理item方法。

## 三、自动备份机制
每次创建或者更新schedule_operation时，都会进入time_trigger的register_operation方法中，从而开启一个协程进行自动备份循环中，
```python
    def _create_green_thread(self, first_run_time, timer):
        func = functools.partial(
            self._trigger_operations,
            trigger_property=self._trigger_property.copy(),
            timer=timer)

        self._greenthread = TriggerOperationGreenThread(
            first_run_time, func)
```
可以看到，最终进入了该方法中，跟自动清理机制一样，创建了TriggerOperationGreenThread实例，通过将自动执行操作函数传递进入，while循环执行。  
下面我们来看下function也就是_trigger_operations方法  
```python
    try:
      self._executor.execute_operation(
      operation_id, now, expect_run_time, window)
    except Exception:
    	LOG.exception("Submit operation to executor failed, operation"
		" id=%s", operation_id)
```
该方法调用了executor的方法，这个executor是在cfg里面设置的
```python
    @property
    def executor(self):
        if not self._executor:
            executor_cls = import_driver.DriverManager(
                'karbor.operationengine.engine.executor',
                cfg.CONF.operationengine.executor).driver
            self._executor = executor_cls(self.operation_manager)
        return self._executor
```
我们进入最终执行的方法里面（green_thread_executor.py）
```python
        param = {
            'operation_id': operation_id,
            'triggered_time': triggered_time,
            'expect_start_time': expect_start_time,
            'window_time': window_time,
            'run_type': constants.OPERATION_RUN_TYPE_EXECUTE
        }
        try:
            self._create_thread(self._run_operation, operation_id, param)
        except Exception:
            self._operation_thread_map.pop(operation_id, None)
            LOG.exception("Execute operation (%s), and create green thread "
                          "failed", operation_id)
```
改方法新建一个协程执行run_operation方法  
run_operation方法会根据operation_type来实例化执行类，然后调用该类的run方法  
这里也就是要讲讲operation的插件化机制  

### operation插件化
在karbor的operationengine下的protection目录，operation_manager管理器会自动加载该目录下的所有operation类，使用map对象持有，自动化执行的时候，会根据执行操作类型自动执行对应operation类的run方法  
注：所以要新加一个执行操作，只需在目录下新建操作类，并实现run方法













