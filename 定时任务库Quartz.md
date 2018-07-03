## 一、Quartz基本概念

1、Job： 是一个接口，只定义一个方法execute(JobExecutionContext context)，在实现接口的execute方法中编写所需要定时执行的Job(任务)， JobExecutionContext类提供了调度应用的一些信息。Job运行时的信息保存在JobDataMap实例中；

在Quartz中，每次Scheduler执行Job时，在调用其execute()方法之前，它需要先根据JobDetail提供的Job类型创建一个Job class的实例，在任务执行完以后，Job class的实例会被丢弃，Jvm的垃圾回收器会将它们回收。因此编写Job的具体实现时，需要注意：  
(1) 它必须具有一个无参数的构造函数；  
(2) 它不应该有静态数据类型，因为每次Job执行完以后便被回收，因此在多次执行时静态数据没法被维护。

I、`@DisallowConcurrentExecution`  
此标记用在实现Job的类上面,意思是不允许并发执行，Job(任务)的执行时间[比如需要10秒]大于任务的时间间隔[Interval（5秒)]，那么默认情况下，调度框架为了能让 任务按照我们预定的时间间隔执行,会马上启用新的线程执行任务。  
加上这个标记就会等待任务执行完毕以后 再重新执行！

II、`@PersistJobDataAfterExecution`   
 此标记说明在执行完Job的execution方法后保存JobDataMap当中固定数据，在默认情况下也就是没有设置 @PersistJobDataAfterExecution的时候，每个job都拥有独立JobDataMap；  
加上该标记时，任务在重复执行的时候具有相同的JobDataMap

2、JobDetail：   
Quartz每次调度Job时， 都重新创建一个Job实例， 所以它不直接接受一个Job的实例，相反它接收一个Job实现类(JobDetail:描述Job的实现类及其它相关的静态信息，如Job名字、描述、关联监听器等信息)，以便运行时通过newInstance()的反射机制实例化Job。

3、Trigger：  
是一个类，描述触发Job执行的时间触发规则。主要有SimpleTrigger和CronTrigger这两个子类。当且仅当需调度一次或者以固定时间间隔周期执行调度，SimpleTrigger是最适合的选择；而CronTrigger则可以通过Cron表达式定义出各种复杂时间规则的调度方案：如工作日周一到周五的15：00~16：00执行调度等；
触发器也有一个和它相关的JobDataMap，它是用来给被触发器触发的job传参数的。

4、Scheduler：  
任务调度器

5、Misfire：  
错过的，指本来应该被执行但是实际没有被执行的任务调度

6、 JobExecutionContext：  
对象提供了带有job运行时的信息：执行它的调度器句柄、触发它的触发器句柄、job的JobDetail对象、JobDataMap和一些其他的项;；当Job的trigger触发时，Job的execute(..)方法就会被调度器调用。该对象自动被传递到这个方法里

7、JobDataMap：  
因为job的执行机制，在job类中，不应该定义有状态的数据属性，因为在job的多次执行中，这些属性的值不会保留；
那么如何给job实例增加属性或配置呢？如何在job的多次执行中，跟踪job的状态呢？答案就是：JobDataMap

8、命名  
Groups是用来组织分类jobs和triggers的，以便今后的维护。在一个组里的job和trigger的名字必须是唯一的，换句话说，一个job和trigger的全名为他们的名字加上组名。如果把组名置为”null”，系统会自动给它置为Scheduler.DEFAULT_GROUP

## 二、Quartz配置文件

在配置文件quartz.properties中增加相关配置

为了使用quartz，一个基本的quartz.properties配置文件如下所示：
```
org.quartz.scheduler.instanceName=scheduler
org.quartz.scheduler.instanceId=AUTO
org.quartz.scheduler.skipUpdateCheck=true
org.quartz.threadPool.class=org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.threadCount=3
org.quartz.threadPool.threadPriority=5
```

上述配置的scheduler有如下特点：
* org.quartz.scheduler.instanceName - scheduler的名称为“MyScheduler”
* org.quartz.scheduler.skipUpdateCheck—检查quartz的版本更新
* org.quartz.threadPool.class  —一个实现了 org.quartz.spi.ThreadPool 接口的类的全限名称
* org.quartz.threadPool.threadCount - 线程池中有3个线程，即最多可以同时执行3个job；
* org.quartz.threadPool.threadPriority—设置工作者线程的优先级。优先级别高的线程比级别低的线程更优先得到执行

## 三、Quartz的启动流程

若quartz是配置在spring中，当服务器启动时，就会装载相关的bean；
SchedulerFactoryBean实现了InitializingBean接口，因此在初始化bean的时候，会执行afterPropertiesSet方法；
该方法将会调用SchedulerFactory(DirectSchedulerFactory 或者 StdSchedulerFactory，通常用StdSchedulerFactory)创建Scheduler；
SchedulerFactory在创建quartzScheduler的过程中，将会读取配置参数，初始化各个组件


比如——  
装载quartz配置如下——schedule.service.xml  
```xml
<bean id="scheduler"
	class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
	<property name="configLocation" value="classpath:quartz.properties" />
</bean>
```

Quartz属性配置如下——quartz.properties  
```
org.quartz.scheduler.instanceName=scheduler
org.quartz.scheduler.instanceId=AUTO
org.quartz.scheduler.skipUpdateCheck=true
org.quartz.threadPool.class=org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.threadCount=10
org.quartz.threadPool.threadPriority=5
```

## 四、实战
实现一个最简单的 Quartz 定时任务，有以下几个步骤：  
* 创建 Job
* 创建 JobDetail 
* 创建 表达式调度构建器CronScheduleBuilder
* 根据调度器创建触发器CronTrigger
* 创建调度器Scheduler
* 将JobDetail 和CronTrigger传入Scheduler，实现定时调度

Quartz API核心接口有：
* Scheduler – 与scheduler交互的主要API；
* Job – 你通过scheduler执行任务，你的任务类需要实现的接口；
* JobDetail – 定义Job的实例；
* Trigger – 触发Job的执行；
* JobBuilder – 定义和创建JobDetail实例的接口;
* TriggerBuilder – 定义和创建Trigger实例的接口；
