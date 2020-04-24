# Openstack定时任务

在介绍karbor的定时任务之前，先看下openstack的定时任务机制

## 定时任务的方式

OpenStack 定时任务实现由两种实现方法，一种是通过 `periodic_task` 函数装饰器， 另外一种是由 `DynamicLoopingCall` 和 `FixedIntervalLoopingCall` 类通过协程来实现。

这两种定时任务的目的也完全不一样，前者一般都是用来装饰 `manager` 类方法，用来实现资源定时刷新、状态报告等；后者通过 `wait()` 调用进行阻 塞，等待某些某些特定事件发生！


### 函数装饰器 
#### 1、operationengine
* protection进程监控
```python
    @periodic_task.periodic_task(spacing=60, run_immediately=True)
    def _engine_process_monitor(self, context):
        LOG.debug("Monitor engine process: %s " % self.protection_process)
        for k in self.protection_process.keys():
            self.protection_process[k] += 1
        dead_engine_process = []
```
* 计量数据上报
```python
   @periodic_task.periodic_task(spacing=CONF.meter_interval_time,
                                 run_immediately=True)
    def _report_meter_data(self, context):
        """
        上报计量数据到ceilometer
        :param context:
        :return:
        """
        if CONF.scene == cbs_constants.SCENE_PRIVATE_CLOUD:
            meter_report.report_meter_data(context)
```

#### 2、protection
protection进程心跳
```python
    @periodic_task.periodic_task(spacing=30, run_immediately=True)
    def _process_notify(self, ctxt):
        LOG.debug(
            "engine_process_notify to operationengine.%s"
            % self.rebuild_manager.process_uuid)
        self.trigger_api.engine_process_notify(
            ctxt,
            self.rebuild_manager.process_uuid)
```

这里就顺便说说protection的心跳机制
#### 进程心跳
* `protection`进程定时发送自己的`uuid`给`operationengine`；
* `operationengine`持有`protection_uuid`字典，将该字典的`uuid`值重置为0；  
* `operationengine`的定时进程监控任务会每执行一次就将其持有的`protection_uuid`字典中所有的`protection_uuid`加1；然后找到所有uuid值超过3的进程id，进入死亡进程处理逻辑（即任务的续作流程）
注：`protection_uuid`字典值超过3则认为该已经3次循环都没收到该`protection`的心跳了

### FixedIntervalLoopingCall类
karbor中很多地方也用了这种方式  
* rabbitmq监控
* plugin插件任务检查完成情况
* ...

