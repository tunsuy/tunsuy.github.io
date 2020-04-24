# taskflow回滚机制

## 一、TaskFlow源码文件简介
Taskflow主要的几个组件是：builder、compiler、runtime、scheduler、executor  
**1. Builder**：  
状态机的编译器。在非`GAME_OVER`和`UNDEFINED`任何状态之间，如果engine暂停或engine发生故障（由于一个不可解决的任务失败或调度失败），engine将停止执行新的任务（当前正在运行的任务将被允许完成），此时这个状态机的运行循环将被打破。  
**2. Compiler**：  
分为task的编译器、flow的递归编译器  
**3. Executor**：  
提交任务进行执行  
**4. Runtime**：  
该对象包含各种实用方法和属性，一个所需的运行时组件和功能的集合  
**5. Scheduler**：  
使用runtime的fetch_scheduler协程安全地调度atoms  

## 二、TaskFlow的状态机回滚机制
Taskflow的入口是在engine的run方法中，run方法又是调用的run_iter生成器方法  
Run_iter方法的主体内容如下：
```python
try:
    closed = False
    machine, memory = self._runtime.builder.build(
    self._statistics, timeout=timeout,
    gather_statistics=self._gather_statistics)
    r = runners.FiniteRunner(machine)
    for transition in r.run_iter(builder.START):
    last_transitions.append(transition)
    _prior_state, new_state = transition
    # NOTE(harlowja): skip over meta-states
    if new_state in builder.META_STATES:
    continue
```
该方法首先调用了builder的build方法生成了一个状态机，进入build方法，可以发现：该方法主要是对各种状态的定义、状态之间的关系、状态的执行事件等，然后通过生成automaon库的machine类实例，调用该实例的方法对状态机进行相关的定义。  
我们拿game_over状态事件来分析下：  
```python
	def game_over(old_state, new_state, event):
      # This reaction function is mainly a intermediary delegation
      # function that analyzes the current memory and transitions to
      # the appropriate handler that will deal with the memory values,
      # it is *always* called before the final state is entered.
      if memory.failures:
      return FAILED
      with self._storage.lock.read_lock():
      leftover_atoms = iter_utils.count(
      # Avoid activating the deciders, since at this point
      # the engine is finishing and there will be no more further
      # work done anyway...
      iter_next_atoms(apply_deciders=False))
      if leftover_atoms:
      # Ok we didn't finish (either reverting or executing...) so
      # that means we must of been stopped at some point...
      LOG.trace("Suspension determined to have been reacted to"
      " since (at least) %s atoms have been left in an"
      " unfinished state", leftover_atoms)
      return SUSPENDED
      elif self._runtime.is_success():
      return SUCCESS
      else:
      return REVERTED
```
 
可以看到，它是调用了runtime的一个is_success方法来判断是否成功或者回滚，而is_success方法又是通过atom_state的状态来判断的
```python
    def is_success(self):
        """Checks if all atoms in the execution graph are in 'happy' state."""
        atoms = list(self.iterate_nodes(com.ATOMS))
        atom_states = self._storage.get_atoms_states(atom.name
                                                     for atom in atoms)
        for atom in atoms:
            atom_state, _atom_intention = atom_states[atom.name]
            if atom_state == st.IGNORE:
                continue
            if atom_state != st.SUCCESS:
                return False
        return True
```
只要不是success和ignore状态的都会进入回滚流程，而atom_state状态的变化完全是由automaton库自己控制的。

## 三、TaskFlow的业务task执行机制

从builder文件构建状态机的schedule事件入手，一层层调用，最终我们进入executor模块中，我们看到，业务task的最终执行是在_execute_task方法中被调用的：
```python
def _execute_task(task, arguments, progress_callback=None):
    with notifier.register_deregister(task.notifier,
                                      ta.EVENT_UPDATE_PROGRESS,
                                      callback=progress_callback):
        try:
            task.pre_execute()
            result = task.execute(**arguments)
        except Exception:
            # NOTE(imelnikov): wrap current exception with Failure
            # object and return it.
            result = failure.Failure()
        finally:
            task.post_execute()
    return (EXECUTED, result)
```
该方法调用了task基类的execute方法，任何业务task都会继承task基类并重写它的execute方法和revert方法。我们发现这里是捕获了业务task的所有异常，并生成一个Failure的失败实例并返回。  
我们回到builder模块的analyze事件方法中，进入complete_an_atom方法：
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
        if isinstance(result, failure.Failure):
        retain = do_complete_failure(atom, outcome, result)
        if retain:
        memory.failures.append(result)
        else:
        # NOTE(harlowja): avoid making any intention request
```
然后会去调用do_complete方法，也就是completer模块的complete方法：
```python
    def complete(self, node, outcome, result):
        """Performs post-execution completion of a node result."""
        handler = self._runtime.fetch_action(node)
        if outcome == ex.EXECUTED:
            handler.complete_execution(node, result)
        else:
            handler.complete_reversion(node, result)
```
 
这里会调用action模块中task的complete_execution方法：
```python
    def complete_execution(self, task, result):
        if isinstance(result, failure.Failure):
            self.change_state(task, states.FAILURE, result=result)
        else:
            self.change_state(task, states.SUCCESS,
                              result=result, progress=1.0)
```
这里就会判断task的result是否是Failure实例对象，从而改变状态机的状态，这就回到了上面讲解状态机的状态变更时的回滚逻辑
