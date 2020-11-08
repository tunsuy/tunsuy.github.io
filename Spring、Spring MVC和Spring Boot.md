# Spring、Spring MVC和Spring Boot

## Spring
Spring Framework是最流行的Java应用程序开发框架。 Spring Framework的主要功能是依赖项注入或控制反转（IoC）。借助Spring Framework，我们可以开发一个松耦合的应用程序。

项目构建：选择New project->Spring。
通过idea的项目构建工程，我们可以知道，它会帮我们把所有的spring依赖库添加进来，其他什么都不会做，也就是说所有的配置项都需要手动添加。


## Spring MVC
Spring MVC是由Spring框架管理并基于Servlet的完整的面向MVC的Http框架。它相当于JavaEE堆栈中的JSF。其中最流行的元素是带有@Controller注释的类，在这个类中可以实现使用不同的HTTP请求访问（GET、POST）的方法。同时它还有一个具有等效的注解@RestController，用来实现基于REST的API。

项目构建：选择New project->Spring，然后勾选Spring MVC
通过idea的项目构建工程，我们可以知道，它除了像上面spring那样给我们添加库之外，还会添加如下一些配置文件

### 1、pom.xml文件
不会自动生成该文件，需要我们自己根据业务需要添加相关依赖。

### 2、web.xml文件
自动帮我们生成了该文件：
```xml
<context-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>/WEB-INF/applicationContext.xml</param-value>
</context-param>
<listener>
	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
<servlet>
	<servlet-name>dispatcher</servlet-name>
	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
	<load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
	<servlet-name>dispatcher</servlet-name>
	<url-pattern>*.form</url-pattern>
</servlet-mapping>
```
另外，还自动生成了如下两个文件：
* dispatcher-servlet.xml：该文件主要作为web请求分发的bean配置文件
* applicationContext.xml: 该文件就是spring配置文件，也就是IOC文件

## Spring Boot
Spring boot是一个用于快速构建应用程序的实用工具，提供开箱即用的配置，以便构建支持Spring的应用程序。Spring boot集成了各种不同的模块，例如spring-core，spring-data，spring-web（顺便说一下，包括Spring MVC）等等。使用spring boot，你可以选择需要的模块，并自动配置它们。
它避免了很多样板代码。它在幕后隐藏了很多复杂性逻辑，因此开发人员可以快速上手并轻松开发基于Spring的应用程序。

项目构建：选择New project->Spring Initializr，然后选择相应的模块
通过idea的项目构建工程，我们可以知道，它为我们做了如下一些事情

### 1、pom.xml文件
```xml
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>2.3.4.RELEASE</version>
	<relativePath/> <!-- lookup parent from repository -->
</parent>

<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-quartz</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
	<dependency>
		<groupId>org.mybatis.spring.boot</groupId>
		<artifactId>mybatis-spring-boot-starter</artifactId>
		<version>2.1.3</version>
	</dependency>

	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
		<exclusions>
			<exclusion>
				<groupId>org.junit.vintage</groupId>
				<artifactId>junit-vintage-engine</artifactId>
			</exclusion>
		</exclusions>
	</dependency>
</dependencies>
```
根据我们在场景项目时选择的功能模块，自动添加了相应了依赖。

### 2、web.xml文件
没有该文件，也就是说spring boot的运行不再依赖该文件，即不再依赖tomcat容器，它自身就集成了一个servlet容器。

### 3、启动文件
帮我们自动生成了一个xxxApplication.java类文件：
```java
@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```
不用任何配置，直接执行该文件，就可以将一个spring boot项目运行起来。

## 对比总结
从上面的分析我们可以看出，Spring和Spring MVC其实没什么区别，Spring MVC只是作为Spring框架项目下的一个子模块：提供了基于MVC的web框架支持，其他的配置和开发流程没有任何区别。

下面则主要对比下Spring 和 Spring Boot，Spring Boot 和Spring MVC
### Spring Boot和Spring MVC
| Spring Boot                                                  | Spring MVC                                |
| ------------------------------------------------------------ | ----------------------------------------- |
| Spring的模块集合，用于使用合理的默认值打包基于Spring的应用程序。 | Spring框架下基于模型视图控制器的Web框架。 |
| 它提供了默认配置来构建Spring支持的框架。                     | 它提供了用于构建Web应用程序的即用型功能。 |
| 无需手动构建配置                                             | 需要手动构建配置                          |
| 它避免了样板代码，并将依赖项包装在一个单元中。 | 它分别指定每个依赖项

### Spring Boot和Spring
| Spring                                                   | Spring Boot                                                  |
| -------------------------------------------------------- | ------------------------------------------------------------ |
| 主要功能是依赖项注入。                                   | 主要功能是自动配置。它会根据需求自动配置类。                 |
| 通过允许我们开发松耦合应用程序，它可以使事情变得更简单。 | 它有助于创建配置更少的独立应用程序。                         |
| 开发人员编写了大量代码（样板代码）来完成最小的任务。     | 它减少了样板代码。                                           |
| 为了测试Spring项目，我们需要显式设置服务器。             | Spring Boot提供了Jetty和Tomcat等嵌入式服务器。               |
| 它不提供对内存数据库的支持。                             | 它提供了几个插件来处理嵌入式和内存数据库（例如H2）。         |
| 开发人员在pom.xml中手动定义Spring项目的依赖项。          | Spring Boot在pom.xml文件中带有启动程序的概念，该文件在内部负责根据Spring Boot Requirement下载依赖项JAR。 |