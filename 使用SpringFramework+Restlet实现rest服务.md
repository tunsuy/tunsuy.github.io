# 使用SpringFramework+Restlet实现rest服务

## 什么是REST
REST 全称是 Representational State Transfer（表述性状态转移），它是 Roy Fielding 博士在 2000 年写的一篇关于软件架构风格的论文。许多知名互联网公司开始采用这种轻量级 Web 服务，大家习惯将其称为 RESTful Web Services，或简称 REST 服务。

REST 本质上是使用 URL 来访问资源的一种方式。总所周知，URL 就是我们平常使用的请求地址了，其中包括两部分：请求方式 与 请求路径，比较常见的请求方式是 GET 与 POST，但在 REST 中又提出了几种其它类型的请求方式，汇总起来有六种：GET、POST、PUT、DELETE、HEAD、OPTIONS。尤其是前四种，正好与 CRUD（增删改查）四种操作相对应：GET（查）、POST（增）、PUT（改）、DELETE（删）。

实际上，REST 是一个“无状态”的架构模式，因为在任何时候都可以由客户端发出请求到服务端，最终返回自己想要的数据。也就是说，服务端将内部资源发布 REST 服务，客户端通过 URL 来访问这些资源，这不就是 SOA 所提倡的“面向服务”的思想吗？所以，REST 也被人们看做是一种轻量级的 SOA 实现技术，因此在企业级应用与互联网应用中都得到了广泛使用。

在 Java 的世界里，有一个名为 JAX-RS 的规范，它就是用来实现 REST 服务的。目前有许多框架已经实现了该规范，比如restlet、cxf。

restlet可以单独使用，也可以与springframework继承一起使用，下面讲解第二种。

## 使用 Spring + restlet 发布 REST 服务
### 添加maven依赖
```xml
<dependencies>
	<!-- Spring -->
	<dependency>
		<groupId>org.springframework</groupId>
		<artifactId>spring-web</artifactId>
		<version>${spring.version}</version>
	</dependency>
	<!-- restlet -->
	<dependency>
		<groupId>org.restlet.jee</groupId>
		<artifactId>org.restlet</artifactId>
	</dependency>
	<dependency>
		<groupId>org.restlet.jee</groupId>
		<artifactId>org.restlet.ext.jackson</artifactId>
		<exclusions>
			<exclusion>
				<groupId>org.codehaus.woodstox</groupId>
				<artifactId>woodstox-core-asl</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
	<dependency>
		<groupId>org.restlet.jee</groupId>
		<artifactId>org.restlet.ext.servlet</artifactId>
	</dependency>
	<dependency>
		<groupId>org.restlet.jee</groupId>
		<artifactId>org.restlet.ext.spring</artifactId>
	</dependency>
</dependencies>
```
这里仅依赖 Spring Web 模块（无需 MVC 模块）。

### 配置web.xml
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

......

<listener>
	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class> 
</listener>
<servlet>
	<servlet-name>restlet</servlet-name>
	<servlet-class>org.restlet.ext.spring.RestletFrameworkServlet</servlet-class>
	<load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
	<servlet-name>restlet</servlet-name>
	<url-pattern>/*</url-pattern>
</servlet-mapping>   
```
使用 Spring 提供的 ContextLoaderListener 去加载所有的配置文件*.xml；使用 restlet 提供的 RestletFrameworkServlet 去处理前缀为所有的 REST 请求。

### 定义Resource资源类
```java
@Controller("backupPolicyCreateAndQuery")
@Scope("prototype")
public class BackupPolicyCreateAndQuery extends ServerResource
{
	@Post
    public BackupPolicyCreateResponse createPolicy(BackupPolicyCreateRequest request)
    {......}
}
```
创建resource代码类，要继承`org.restlet.resource.ServerResource`，用@Get和@Post注解来什么类型的请求

### 配置spring文件
下面只举一个示例，spring配置相信大家都比较熟悉了，就不多说
```xml
<context:component-scan base-package="com.huawei.hwclouds.autobackup"/>
<context:annotation-config />
	
<bean id="scheduleConfigs"
	class="org.springframework.beans.factory.config.PropertiesFactoryBean">
	<property name="locations">
		<list>
			<value>classpath:service.config.properties</value>
		</list>
	</property>
</bean> 

<bean id="scheduler"
	class="org.springframework.scheduling.quartz.SchedulerFactoryBean" autowire="no">
	<property name="configLocation" value="classpath:quartz.properties" />
</bean>

......
```

### 将接口的实现类发布为SpringBean
有两种方式：一是使用spring配置文件；一是使用注解。这里采用第二种，如上面定义Resource资源类所示：

### 创建restlet-servlet.xml
在web.xml同目录下创建{name}-servlet.xml，其中name为web.xml中定义的servlet名称
```xml
<bean name="root" class="org.restlet.ext.spring.SpringRouter">
	<property name="attachments">
		<map>
			<!-- 此处可以按业务模块划分url v1 or v2 -->
			<entry key="/v1/{tenant}" value-ref="v1Route" />
			<entry key="/v2/{tenant}" value-ref="v2Route" />
			
			<entry key="/server_status">
				<bean class="org.restlet.ext.spring.SpringFinder">
					<lookup-method name="create" bean="autoBackupStatusResource" />
				</bean>
			</entry>
		</map>
	</property>
</bean>

<bean id="v1Route" class="org.restlet.ext.spring.SpringRouter">
	<property name="attachments">
		<map>
			<entry key="/backuppolicy">
				<bean class="org.restlet.ext.spring.SpringFinder">
					<lookup-method name="create"
						bean="backupPolicyCreateAndQuery" />
				</bean>
			</entry>
		</map>
	</property>
</bean>
```
这个文件主要是配置url的路径对应的资源bean，可以看到，这里的bean也是可以引用的。