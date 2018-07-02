# taskflow工作机制
## 一、Taskflow的几个主要单位
### 1、Engine
Engine是taskflow的启动入口， 主要工作是创建状态机，循环状态机，在指定状态下通过 Executor执行任务。Engine分为好几种：work based 的 Engine 比较特殊我们不看，直接看 action engine。  
几种action engine 其实没有什么区别，通过 Executor 分为  
I、序列化（阻塞，顺序执行）引擎  
II、基于线程的并行引擎  
III、基于协程的并行引擎  
IV、基于多进程的并行引擎  
Engines是真正运行 Atoms 的对象，用于 load 一个 flow，然后驱动这个 flow 中的 task/flow 开始运行. 我们可以通过 engine_conf 参数来指定不同的 engine 类型

### 2、Executor 
前面说了，Engine会通过Executor执行任务，因为如果Engine直接执行任务的话，整个状态机的循环会受到正在执行的任务的影响，所以包了一层Executor来执行具体的任务(当然具体代码里对Executor 的应用会更复杂一点，为了扩展和异常处理包了3层) 。  
在taskflow的源代码中Executor是通过futurist库来实现的，而futurist又是基于futures的。  
具体的任务代码在一般情况下可以不处理异常，因为执行任务的代码通过 except Exception 捕获了任务的所有异常。  
特殊异常就是 CancelledError，这个异常是调度到已经取消任务时由 futurist 抛出，在读代码的时候需注意下这个的特殊处理 。  

### 3、Scheduler 
这个没什么好说的，Executor 的封装的最上层，最后执行会落实到具体的 Executor 上 
### 4. storage 
这个是存储接口，后面说到 flow 的时候会详细讲到，storage 的初始化在 Engine 中，一个功能是数据存储的接口，一个功能是作为 flow 的外层封装 
### 5. Runtime 与 machine 
machine 就是 Engine 中循环的(automaton)状态机了，一个 engine 只运行一个状态机，初始化代码在 builder.MachineBuilder，MachineBuilder 又是在 Runtime 中调用生成 machine 的，我们先别管 Runtime，先理解一下 taskflow 的状态机   
其他概念：  
I、**Atom** 是 TaskFlow 的最小单位，其他的所有类，包括 Task 类都是 Atom 类的子类.  
II、**Task** 是拥有执行和回滚功能额最小工作单元. 在 Task 类中开发者能够自定义 execute(执行) 和 revert(回滚) method.  
III、**Flow**是 TaskFlow 中使用来关联各个 Task 的，并且规定这些 Task 之间的执行和回滚顺序. flow 中不仅能够包含 task 还能够嵌套 flow. 常见类型有以下几种:  
- **Linear(linear_flow.Flow)**: 线性流， 该类型 flow 中的 task/flow 按照加入的顺序来依次执行， 按照加入的倒序依次回滚.
- **Unordered(unordered_flow.Flow)**: 无序流， 该类型 flow 中的 task/flow 可能按照任意的顺序来执行和回滚.
- **Graph(graph_flow.Flow)**: 图流， 该类型 flow 中的 task/flow 按照显式指定的依赖关系或通过其间的 provides/requires 属性的隐含依赖关系来执行和回滚.  
IV、**Retry** 是一个控制当错误发生时， 如何进行重试的特殊工作单元， 而且当你需要的时候还能够以其他参数来重试执行别的 Atom 子类. 

## 二、TaskFlow中flow怎么执行task
来看看Task的基类atom中的_init__， 它调用的**atom.py**最上面的_build_arg_mapping 和_build_rebind_dict，这两个是非常关键的函数。  
初始化的时候，通过反射execute函数获取到execute的参数列表及参数对应的默认值 有默认值的为可选参数，没有默认值的是必要参数，存入一个有序字典中  
flow通过记录这个有序字典，当有对应参数出现（由前一个任务生成）的时候，就会调用这个task。  
也就是说，对于在flow中的atask如果发现有a， b参数生成（c是必要参数），那么atask会被调用 （这里只是简单描述下flwo的调用，实际调用会比较复杂）

```python
    self.rebind, exec_requires, self.optional = self._build_arg_mapping(
    self.execute,
    requires=requires,
    rebind=rebind, auto_extract=auto_extract,
    ignore_list=ignore_list
    )

    revert_ignore = ignore_list + list(_default_revert_args)
    revert_mapping = self._build_arg_mapping(
    self.revert,
    requires=revert_requires or requires,
    rebind=revert_rebind or rebind,
    auto_extract=auto_extract,
    ignore_list=revert_ignore
    )
    (self.revert_rebind, addl_requires,
    self.revert_optional) = revert_mapping

    self.requires = exec_requires.union(addl_requires)
```
 
**Task参数简介**：  
I、**rebind**：用于生成的参数名映射 而flow也能通过映射的参数来调用task  
II、**provides**：前面我们说rebind的时候有提过上前一个任务提供参数，provides就是表示当前任务的execute的执行结果能提供什么参数。默认情况下provides为None，也就是说无论execute的执行结果是什么，当前task都不提供任何参数到外部  
III、**inject**：一个不可变的name->value字典， 他指定了必须在atome执行之前自动注入到当前atom区域的初始化输入值。通过这种方式允许提供给atom一个不需要提供给其他atom/dependents的local本地值  
IV、**requires**：一个set/list， 用于指定execute method运行必要的输入  

**以创建复制的on_main任务为例说明：**  
通过**dj_manager**的入口文件找到**copy**方法
通过创建flow的层层方法往下看，进入**dj_copy_resource_flow**文件中的_create_hook_task方法中。
```python
    injects = {
    'context': self.context,
    'parameters': parameters,
    'resource': resource,
    'checkpoint': checkpoint
    }
    requires = list(injects)
    requires.append('checkpoint')
    requires.extend(OPERATION_EXTRA_ARGS.get(self.operation_type, []))

    # catch all the exception triggered by method
    def __(*args, **kwargs):
    try:
    method(*args, **kwargs)
    except Exception as err:
    LOG.exception(err)

    task = self.workflow_engine.create_task(__,
    name=task_name,
    inject=injects,
    requires=requires)
    return task

```
 
这里将inject和requires参数传递进workflow文件的TaskFlowEngine类的create_task方法
```python
	def create_task(self, function, requires=None, provides=None,
                    inject=None, **kwargs):
        name = kwargs.get('name', None)
        auto_extract = kwargs.get('auto_extract', True)
        rebind = kwargs.get('rebind', None)
        revert = kwargs.get('revert', None)
        version = kwargs.get('version', None)
        if function:
            return task.FunctorTask(function,
                                    name=name,
                                    provides=provides,
                                    requires=requires,
                                    auto_extract=auto_extract,
                                    rebind=rebind,
                                    revert=revert,
                                    version=version,
                                    inject=inject)
```
 
该方法最终就实例化了taskflow框架的FunctorTask类，并将inject和requires参数初始化给它。

## 三、TaskFlow的状态变化机制
1、在engine入口文件中，通过一些执行前编译、准备、检查等操作，初始化了一些实例变量，其中就有self._runtime.  
2、调用builder的build方法
```python
	try:
      closed = False
      machine, memory = self._runtime.builder.build(
      self._statistics, timeout=timeout,
      gather_statistics=self._gather_statistics)
      r = runners.FiniteRunner(machine)
```
 
在该方法中会定义几个变量分别引用completer模块的几个完成方法
```python
    # Cache some local functions/methods...
    do_complete = self._completer.complete
    do_complete_failure = self._completer.complete_failure
    get_atom_intention = self._storage.get_atom_intention
```
 
然后再complete_an_atom方法中进行调用，并将结果进行传入，它会捕获task的异常
```python
 	def complete_an_atom(fut):
      # This completes a single atom saving its result in
      # storage and preparing whatever predecessors or successors will
      # now be ready to execute (or revert or retry...); it also
      # handles failures that occur during this process safely...
      atom = fut.atom
      try:
      outcome, result = fut.result()
      do_complete(atom, outcome, result)
```
 
在completer模块的complete方法中通过传入的result，进行执行是否回滚
```python
    def complete(self, node, outcome, result):
        """Performs post-execution completion of a node result."""
        handler = self._runtime.fetch_action(node)
        if outcome == ex.EXECUTED:
            handler.complete_execution(node, result)
        else:
            handler.complete_reversion(node, result)
```

