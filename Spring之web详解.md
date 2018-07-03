
## 一、Tomcat

因为web最终是运行在类似于Tomcat这种web容器里面的，它对于web的生命周期应该有着直接的关系， 下面是Tomcat的容器模型：  
![https://github.com/tunsuy/tunsuy.github.io/blob/master/images/tomcat.jpg](https://github.com/tunsuy/tunsuy.github.io/blob/master/images/tomcat.jpg)

从上图可以看出 Tomcat 的容器分为四个等级，真正管理 Servlet 的容器是 Context 容器，一个 Context 对应一个 Web 工程。尤其是后面一句话，一个 Context 对应一个 Web 工程，所以我们在Tomcat根目录的webapps文件夹路径下面经常会看到除了我们自己部署的web，还有若干其他Tomcat自带的web，不同的web工程都会对应在Tomcat里面的context容器。

Tomcat的启动是从顶层开始一直到到Engine到Host再到StandardContext（这里的context我可以理解为就是对应每个web工程的容器了）， 当 Context 容器初始化状态设为 init 时，添加在 Context 容器的 Listener 将会被调用。ContextConfig 继承了 LifecycleListener 接口，ContextConfig 类会负责整个 Web 应用的配置文件的解析工作。

## 二、web应用的初始化

1、Web 应用的初始化工作是在 ContextConfig 的 configureStart 方法中实现的；

2、应用的初始化主要是要解析 web.xml 文件，这个文件描述了一个 Web 应用的关键信息，也是一个 Web 应用的入口。

3、web.xml 文件中的各个配置项将会被解析成相应的属性保存在 WebXml 对象中，接下去将会将 WebXml 对象中的属性设置到 Context 容器中，这里包括创建 Servlet 对象、filter、listener 等等，所有 web.xml 属性都被解析到 Context 中。

所以说 Context 容器才是真正运行 Servlet 的 Servlet 容器。一个 Web 应用对应一个 Context 容器，容器的配置属性由应用的 web.xml 指定

## 三、web.xml配置文件

web.xml中配置有ContextLoaderListener，也可以自定义一个实现了ServletContextListener接口的Listener类，web.xml中的配置举例如下：
```xml
<listener>
	    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class> 
</listener>
```

Web容器在启动的过程中会触发ServletContextEvent事件，会被ContextLoaderListener监听到，并调用ContextLoaderListener中的contextInitialized方法；

ContextLoaderListener类继承了ContextLoader，在初始化Context的过程中，调用ContextLoader的initWebApplicationContext方法初始化WebApplicationContext；

WebApplicationContext是一个接口，Spring默认的实现类为XmlWebApplicationContext，XmlWebApplicationContext就是Spring的IoC容器；

在初始化XmlWebApplicationContext之前，Web容器已经加载了context-param。作为Spring的IoC容器，其对应的Bean定义的配置正是context-param指定的：

## 四、项目使用流程

以Tomcat为例，想在Web容器中使用Spirng MVC，必须进行四项的配置：
* 修改web.xml；
* 添加servlet定义
* 编写servletname-servlet.xml（ servletname是在web.xm中配置Servlet时的servlet-name的值）
* 配置filter过滤器
* 配置contextConfigLocation初始化参数
* 配置ContextLoaderListerner。
* 根据业务情况分类编写定义各种bean类的xml文件
* 将所有xml文件引入web.xml中

## 五、实战
1、web.xml文件  
1.1 打开web.xml文件：autobackup-server\src\main\webapp\WEB-INF\web.xml  
1.2 servlet定义如下：
```xml
<servlet>
	<servlet-name>restlet</servlet-name>
	<servlet-class>org.restlet.ext.spring.RestletFrameworkServlet</servlet-class>
	<load-on-startup>1</load-on-startup>
</servlet>
```
可以看到servlet-name为restlet，那么肯定还存在一个servlet-servlet.xml这样的文件。  
1.3 contextConfigLocation定义如下：
```xml
<context-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>
			  classpath*:META-INF/spring/*.service.xml
		      /WEB-INF/classes/*.datasource.xml 
		      /WEB-INF/classes/*.service.xml
		      classpath*:conf/*.service.xml
		      classpath*:*.service.xml	
	</param-value>
</context-param>
```
这些配置指定Spring IoC容器需要读取的定义了非web层的Bean（DAO/Service）的XML文件路径。  
1.4 listener配置如下：
```xml
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class> 
</listener>
```
配置了Spring MVC在Web容器中的启动类ContextLoaderListener，负责Spring IoC容器在Web上下文中的初始化。

2、restlet-servlet.xml  
2.1 打开restlet-servlet.xml文件：autobackup-server\src\main\webapp\WEB-INF\restlet-servlet.xml  
servlet-servlet.xml定义了WebAppliactionContext上下文中的bean。  
2.2 部分配置如下：  
定义了一个叫v1Route的bean  
`<bean id="v1Route" class="org.restlet.ext.spring.SpringRouter">  `
该bean有个属性字段attachments，是map类型的  
		`<property name="attachments">`  
我们打开该类，可以看到确实有一个实例变量attachments，该变量是一个字典类型的。  
其中一个键值对定义如下：
```xml
    <entry key="/backuppolicy">
		<bean class="org.restlet.ext.spring.SpringFinder">
			<lookup-method name="create"
				bean="backupPolicyCreateAndQuery" />
		</bean>
	</entry>
```
2.2.2.1 这里有个lookup-method，介绍如下  
这个表示spring的查找方法注入。表示查找SpringFinder类中的create方法，将该方法返回的实例用bean="backupPolicyCreateAndQuery"这个bean定义的类来替换。  
我们打开SpringFinder方法，确实看到有一个create方法，返回ServerResource类型实例；  
我们再打开BackupPolicyCreateAndQuery类，可以看到该类确实是继承与ServerResource类。
