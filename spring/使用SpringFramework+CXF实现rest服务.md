# 使用SpringFramework+CXF实现rest服务

## 什么是REST
REST 全称是 Representational State Transfer（表述性状态转移），它是 Roy Fielding 博士在 2000 年写的一篇关于软件架构风格的论文。许多知名互联网公司开始采用这种轻量级 Web 服务，大家习惯将其称为 RESTful Web Services，或简称 REST 服务。

REST 本质上是使用 URL 来访问资源的一种方式。总所周知，URL 就是我们平常使用的请求地址了，其中包括两部分：请求方式 与 请求路径，比较常见的请求方式是 GET 与 POST，但在 REST 中又提出了几种其它类型的请求方式，汇总起来有六种：GET、POST、PUT、DELETE、HEAD、OPTIONS。尤其是前四种，正好与 CRUD（增删改查）四种操作相对应：GET（查）、POST（增）、PUT（改）、DELETE（删）。

实际上，REST 是一个“无状态”的架构模式，因为在任何时候都可以由客户端发出请求到服务端，最终返回自己想要的数据。也就是说，服务端将内部资源发布 REST 服务，客户端通过 URL 来访问这些资源，这不就是 SOA 所提倡的“面向服务”的思想吗？所以，REST 也被人们看做是一种轻量级的 SOA 实现技术，因此在企业级应用与互联网应用中都得到了广泛使用。

在 Java 的世界里，有一个名为 JAX-RS 的规范，它就是用来实现 REST 服务的。目前有许多框架已经实现了该规范，比如restlet、cxf。

cxf可以单独使用，也可以与springframework继承一起使用，下面讲解第二种。

## 使用 Spring + CXF 发布 REST 服务
### 添加maven依赖
```xml
<dependencies>
	<!-- Spring -->
	<dependency>
		<groupId>org.springframework</groupId>
		<artifactId>spring-web</artifactId>
		<version>${spring.version}</version>
	</dependency>
	<!-- CXF -->
	<dependency>
		<groupId>org.apache.cxf</groupId>
		<artifactId>cxf-rt-frontend-jaxrs</artifactId>
		<version>${cxf.version}</version>
	</dependency>
	<!-- Jackson -->
	<dependency>
		<groupId>com.fasterxml.jackson.jaxrs</groupId>
		<artifactId>jackson-jaxrs-json-provider</artifactId>
		<version>${jackson.version}</version>
	</dependency>
</dependencies>
```
这里仅依赖 Spring Web 模块（无需 MVC 模块），此外就是 CXF 与 Jackson 了。

### 配置web.xml
```xml
<!-- Spring -->
<context-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>classpath:spring.xml</param-value>
</context-param>
<listener>
	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>

<!-- CXF -->
<servlet>
	<servlet-name>cxf</servlet-name>
	<servlet-class>org.apache.cxf.transport.servlet.CXFServlet</servlet-class>
</servlet>
<servlet-mapping>
	<servlet-name>cxf</servlet-name>
	<url-pattern>/ws/*</url-pattern>
</servlet-mapping>
```
使用 Spring 提供的 ContextLoaderListener 去加载 Spring 配置文件 spring.xml；使用 CXF 提供的 CXFServlet 去处理前缀为 /ws/ 的 REST 请求。

### 定义Resource资源类
```java
public interface IBackupVaultRestService
{
    @GET
    @Path("/{siteId}/backup_vaults")
    @Produces("application/json")
    @Consumes("application/json")
    String getBackupVaults(@PathParam("siteId") String siteId,
                           @QueryParam("limit") int limit,
                           @QueryParam("resourceType") String resourceType,
                           @QueryParam("protectType") String protectType,
                           @QueryParam("cloudType") String cloudType,
                           @QueryParam("startPage") int startPage);
}

public class BackupVaultRestServiceImpl extends AbstractRestService implements
        IBackupVaultRestService
{......}
```

### 配置spring文件
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:jaxws="http://cxf.apache.org/jaxws"
	   xmlns:jaxrs="http://cxf.apache.org/jaxrs" xmlns:cxf="http://cxf.apache.org/core"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
http://cxf.apache.org/jaxws http://cxf.apache.org/schemas/jaxws.xsd 
http://cxf.apache.org/jaxrs http://cxf.apache.org/schemas/jaxrs.xsd
http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

	<!-- 发布配置添加的文件 -->
	<import resource="classpath:META-INF/cxf/cxf.xml" />
	<!--<import resource="classpath:META-INF/cxf/cxf-extension-soap.xml" />-->
	<import resource="classpath:META-INF/cxf/cxf-servlet.xml" />
	
	<!-- 安全访问认证拦截器 -->
	<bean id="securityInterceptor"
		  class="com.tunsuy.rest.interceptor.RestSecurityInterceptor" />
		  
	<bean id="backupVaultRestService"
		  class="com.tunsuy.rest.service.platform.cloudresource.BackupVaultRestServiceImpl" />

	<jaxrs:server id="siteServiceContainer" address="/sites">
		<jaxrs:inInterceptors>
			<ref bean="securityInterceptor" />
		</jaxrs:inInterceptors>
		<jaxrs:providers>
			<bean class="com.fasterxml.jackson.jaxrs.json.JacksonJaxbJsonProvider" />
		</jaxrs:providers>
		<jaxrs:serviceBeans>
			<ref bean="backupVaultRestService"/>
		</jaxrs:serviceBeans>
	</jaxrs:server>

```
这里加载了另一个配置文件 cxf.xml，其中包括了关于 CXF 的相关配置。
另外，这里我们可以看到<jaxrs></jaxrs>的标签配置：
这是使用了 CXF 提供的 Spring 命名空间来配置 Service Bean（即上文提到的 Resource Class）与 Provider。注意，这里配置了一个 address 属性为“/sites”，表示 REST 请求的相对路径，与 web.xml 中配置的“/ws/*”结合起来，最终的 REST 请求根路径是“/ws/sites”，在 IBackupVaultRestService 接口方法上 @Path 注解所配置的路径只是一个相对路径。

### 将接口的实现类发布为SpringBean
有两种方式：一是使用spring配置文件；一是使用注解。这里采用第一种，如上面spring文件所示：
```xml
<bean id="backupVaultRestService"
		  class="com.huawei.ism.drm.rest.service.platform.cloudresource.BackupVaultRestServiceImpl" />
```