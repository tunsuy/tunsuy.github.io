## 一、相关概念
#### 1、Runnable
Runnable只有一个run()函数，用于将耗时操作写在其中，该函数没有返回值。然后使用某个线程去执行该runnable即可实现多线程，Thread类在调用start()函数后就是执行的是Runnable的run()函数。 Runnable一般提交给Thread来包装下  ，直接启动一个线程来执行

#### 2、Callable
Callable与Runnable的功能大致相似，Callable中有一个call()函数，但是call()函数有返回值，而Runnable的run()函数不能将结果返回给客户程序。 Callable一般都是提交给ExecuteService来执行

#### 3、Future
Future就是对于具体的Runnable或者Callable任务的执行结果进行取消、查询是否完成、获取结果、设置结果操作。get方法会阻塞，直到任务返回结果。 Executor是Runnable和Callable的调度容器（`future = executor.submit(runnable/callable)，future.get()` 等等）

#### 4、FutureTask
FutureTask则是一个RunnableFuture<V>，而RunnableFuture实现了Runnbale又实现了Futrue<V>这两个接口， 另外它还可以包装Runnable(实际上会转换为Callable)和Callable  <V>，所以一般来讲是一个符合体了，它可以通过Thread包装来直接执行，也可以提交给ExecuteService来执行 ，并且还可以通过v get()返回执行结果，在线程体没有执行完成的时候，主线程一直阻塞等待，执行完则直接返回结果。 `executor.submit(futureTask)， futureTask.get()`

## 二、线程池ThreadPoolExecutor
java的线程池支持主要通过ThreadPoolExecutor来实现，我们使用的ExecutorService的各种线程池策略都是基于ThreadPoolExecutor实现的。

调用ThreadPoolExecutor的execute提交线程：  
1、首先检查CorePool，如果CorePool内的线程小于CorePoolSize，新创建线程执行任务。  
2、如果当前CorePool内的线程大于等于CorePoolSize，那么将线程加入到BlockingQueue（等待执行）。  
3、如果不能加入BlockingQueue（队列满），在小于MaxPoolSize的情况下创建线程执行任务。  
4、如果线程数大于等于MaxPoolSize，那么执行拒绝策略。

Execute和submit的区别：  
submit方法最终会调用execute方法来进行操作，只是他提供了一个Future来托管返回值的处理而已，当你调用需要有返回值的信息时，你用它来处理是比较好的；这个Future会包装对Callable信息，并定义一个Sync对象（），当你发生读取返回值的操作的时候，会通过Sync对象进入锁，直到有返回值的数据通知

执行流程：  
1、当有任务进入时，线程池创建线程去执行任务，直到核心线程数满为止  
2、核心线程数量满了之后，任务就会进入一个缓冲的任务队列中  
当任务队列为无界队列时，任务就会一直放入缓冲的任务队列中，不会和最大线程数量进行比较  
当任务队列为有界队列时，任务先放入缓冲的任务队列中，当任务队列满了之后，才会将任务放入线程池，此时会与线程池中最大的线程数量进行比较，如果超出了，则默认会抛出异常。然后线程池才会执行任务，当任务执行完，又会将缓冲队列中的任务放入线程池中，然后重复此操作。

SpringFrameWork 的 ThreadPoolTaskExecutor 是辅助 JDK 的 ThreadPoolExecutor 的工具类，它将属性通过 JavaBeans 的命名规则提供出来，方便进行配置。 spring包装了一下jdk，其实底层都是jdk的线程池

比如：  
在Spring配置文件中配置——schedule.service.xml
```xml
<bean id="taskExecutor"
	class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">
	<property name="corePoolSize" value="${autobackup.threadPool.corePoolSize}" />
	<property name="keepAliveSeconds" value="${autobackup.threadPool.keepAliveSeconds}" />
	<property name="maxPoolSize" value="${autobackup.threadPool.maxPoolSize}" />
	<property name="queueCapacity" value="${autobackup.threadPool.queueCapacity}" />
</bean>
```

* corePoolSize--线程组保留的最小线程数，如果线程组中的线程数少于此数目，则创建  
* maxPoolSize -线程组中最大线程数  
* keepAliveSeconds -线程组中线程最大不活动时间，则清除此线程  
* queueCapacity ---队列中，用于存放提交的任务


## 三、Java多线程同步阻塞

问：主线程如何等待子线程工作完成呢？  
答：1、 Thread 类提供了join 系列的方法, 这些方法的目的就是阻塞等待当前线程的die.

2、java并发包中的Future也可以实现。future是一个任务执行的结果, 他是一个将来时, 即一个任务执行, 立即异步返回一个Future对象, 等到任务结束的时候, 会把值返回给这个future对象里面，调用future接口的get方法, 会同步等待该future执行结束, 然后获取到结果。

3、使用阻塞队列BlockingQueue 

## 四、ThreadLocal
ThreadLocal是一个关于创建线程局部变量的类。  
通常情况下，我们创建的变量是可以被任何一个线程访问并修改的。而使用ThreadLocal创建的变量只能被当前线程访问，其他线程则无法访问和修改。


实际上ThreadLocal的值是放入了当前线程的一个ThreadLocalMap实例中，所以只能在本线程中访问，其他线程无法访问。

使用场景  
* 实现单个线程单例以及单个线程上下文信息存储，比如交易id等
* 实现线程安全，非线程安全的对象使用ThreadLocal之后就会变得线程安全，因为每个线程都会有一个对应的实例
* 承载一些线程相关的数据，避免在方法中来回传递参数



