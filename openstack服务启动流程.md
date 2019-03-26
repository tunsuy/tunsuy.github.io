* * * ### 启动流程
      加载配置文件；
      设置日志模块；
      初始化持久化对象(oslo_versionedobjects机制)
      启动wsgi服务(web服务)：
      * 加载自定义wsgi的app(oslo_service.Loader加载api-paste.ini文件)
      * 加载manager文件；
      * manager文件：定义一些开放接口，以便于一些服务启动后需要立即执行的，就可以实现该接口。

      #### 问：怎么加载app
      编写api-paste.ini文件
      通过python的Paste.deploy加载机制
      https://tommylike.github.io/openstack/2016/08/03/Paste-Deploy-Introduce/
      http://www.choudan.net/2013/07/28/OpenStack-paste-deploy%E4%BB%8B%E7%BB%8D.html

      #### 问：怎么封装一个周期性任务框架
      参考`oslo_service.periodic_task.py`实现
      使用技术：装饰器、元类、`loopingcall.FixedIntervalLoopingCall`
      通过在元类中获取所有使用了任务装饰器的方法，采用loop进行周期性调度
      对外只需暴露该装饰器方法即可

      #### 问：wsgi服务启动过程
      `launcher.launch_service`会将第一步创建的服务启动起来，然后调用`_start_child`方法。
      在`_start_child`方法中，会调用`os.fork`接口创建子进程，创建的进程数由`launch_service`的workers参数确定，目前默认为1个进程。
      在子进程启动后，调用`_child_process`进行服务启动，调用common中的`launch_service`，此过程主要将service添加到线程池中，并启动。在启动时，会回调`run_service`进而调用`Service.start`方法。

      #### 问：oslo.service是怎样封装服务启动的
      <big>Service类:</big> 描述一个服务，定义一套管理服务生命周期的方法，包括启动服务start()、停止服务stop()、等待服务wait()、重置服务reset()等管理方法。另外，在实例化Service对象时，可以传入一个threads的参数创建一个包含threads个线程的线程组ThreadGroup对象，用来管理服务中的所有线程。
      <big>Services类:</big> 管理一组服务。定义run_service()、stop()、wait()、restart()等生命周期管理方法，定义一个add()方法用于为Services对象添加服务。
      <big>Launcher类:</big> 主要用来启动一个或多个服务并等待其完成。定义`launcher_service(service, workers)、stop()、wait()、restart()`方法管理Launcher对象中所有服务的生命周期；在实例化Launcher对象时，需要传入两个参数conf和restart_method，其中，conf表示加载的服务配置文件，restart_method表示重启服务的方式，在oslo.service中，定义了两种服务重启方式：reload方式表示重启服务时重新加载配置文件，mutate则表示重启服务时改变服务配置文件。
      在oslo.service中，Launcher类主要有两个实现方案：ServiceLauncher类和ProcessLauncher类。其中，ServiceLauncher类是在一个一个父进程中启动一个或多个服务，其通过一个单实例的SignalHandler类监听针对服务的信号。而ProcessLauncher类则是在给定数量的worker中运行一个或多个服务，其启动服务时，首先将服务、workers封装为ServiceWrapper对象；然后，调用ProcessLauncher对象的_start_child()方法启动一个子进程，并在子进程中运行服务。
      <big>launch()方法:</big> 在其他OpenStack组件的实际使用中，通常都会调用oslo_service.service模块下的launch()方法启动相应的服务。该方法需要传入一下几个参数：
      * conf：对应服务的配置参数对象，即oslo.config中的ConfigOpt对象；
      * service：一个实例化的Service对象；
      * workers：为一个服务启动的worker数，如果workers为空或为1，则使用ServiceLauncher启动服务，如果workers>1，则使用ProcessLauncher类启动服务；
      * restart_method：服务重启方式。
        launch()方法会根据上述给定的参数创建服务启动方法类，并调用其launcher_service()方法启动服务。