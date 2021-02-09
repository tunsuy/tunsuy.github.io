# 关于Spring体系的各种启动流程

在介绍spring的启动之前，先来说下启动过程中使用到的几个类
## 基本组件
* 1、BeanFactory：spring底层容器，定义了最基本的容器功能，注意区分FactoryBean
* 2、ApplicationContext：扩展于BeanFactory，拥有更丰富的功能。例如：添加事件发布机制、父子级容器，一般都是直接使用ApplicationContext。
* 3、Resource：bean配置文件，一般为xml文件。可以理解为保存bean信息的文件。
* 4、BeanDefinition：beandifinition定义了bean的基本信息，根据它来创造bean

## 基础流程

不管是哪种系列的spring（springframework、springmvc、springboot、springcloud），Spring的启动过程主要可以分为两部分：
* 第一步：解析成BeanDefinition：将bean定义信息解析为BeanDefinition类，不管bean信息是定义在xml中，还是通过@Bean注解标注，都能通过不同的BeanDefinitionReader转为BeanDefinition类，将BeanDefinition向Map中注册 Map<name,beandefinition>。
  这里分两种BeanDefinition，RootBeanDefintion和BeanDefinition。RootBeanDefinition这种是系统级别的，是启动Spring必须加载的6个Bean。BeanDefinition是我们定义的Bean。
* 第二步：参照BeanDefintion定义的类信息，通过BeanFactory生成bean实例存放在缓存中。
  这里的BeanFactoryPostProcessor是一个拦截器，在BeanDefinition实例化后，BeanFactory生成该Bean之前，可以对BeanDefinition进行修改。
  BeanFactory根据BeanDefinition定义使用反射实例化Bean，实例化和初始化Bean的过程中就涉及到Bean的生命周期了，典型的问题就是Bean的循环依赖。接着，Bean实例化前会判断该Bean是否需要增强，并决定使用哪种代理来生成Bean。

## Springframework
#### 1、容器类
在一般性的spring项目中，大家应该也都知道，一般是通过直接实例化applicationContext类，来实现项目的启动
下面我们来看下通过注解的方式来启动的情况，注解容器定义如下：
```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
	this();
	register(componentClasses);
	refresh();
}

public AnnotationConfigApplicationContext() {
	this.reader = new AnnotatedBeanDefinitionReader(this);
	this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```
创建了注解定义bean读取器和配置文件定义bean扫描器

#### 2、注解定义bean读取器
进入该类构造器中，可以看到最终会执行该方法：
```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
		BeanDefinitionRegistry registry, @Nullable Object source) {

	DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
	if (beanFactory != null) {
		if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
			beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
		}
		if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
			beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
		}
	}

	Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

	if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
		RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
		def.setSource(source);
		beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
	}

	if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
		RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
		def.setSource(source);
		beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
	}
	...
}
```
注册了6个RootBeanDefinition，即系统级别的BeanDefinition。
同时，经过调用`registerPostProcessor->registerBeanDefinition`，可以看到注册BeanDefinition其实就是放到BeanFactory的缓存中。
```java
DefaultListableBeanFactory.java类中

 public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) throws BeanDefinitionStoreException {
      ...
	  this.beanDefinitionMap.put(beanName, beanDefinition);
	  ...
 }
```

上面的6个beanDefinition的实例参数中都有一个postprocessor后缀的类，我们分别点击进入查看即继承关系，可以看到，最终都继承自``接口
```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry var1) throws BeansException;
}

@FunctionalInterface
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory var1) throws BeansException;
}
```

#### 3、BeanFactoryPostProcessor
1、BeanFactoryPostProcessor是spring初始化bean的扩展点。
官文翻译如下：允许自定义修改应用程序上下文的bean定义，调整上下文的基础bean工厂的bean属性值。应用程序上下文可以在其bean定义中自动检测BeanFactoryPostProcessor bean，并在创建任何其他bean之前先创建BeanFactoryPostProcessor。BeanFactoryPostProcessor可以与bean定义交互并修改bean定义，但绝不能与bean实例交互。这样做可能会导致bean过早实例化，违反容器并导致意外的副作用。如果需要bean实例交互，请考虑实现BeanPostProcessor。实现该接口，可以允许我们的程序获取到BeanFactory，从而修改BeanFactory，可以实现编程式的往Spring容器中添加Bean。

也就是说，我们可以通过实现BeanFactoryPostProcessor接口，获取BeanFactory，操作BeanFactory对象，修改BeanDefinition，但不要去实例化bean。

2、BeanDefinitionRegistryPostProcessor是BeanFactoryPostProcessor的子类，在父类的基础上，增加了新的方法，允许我们获取到BeanDefinitionRegistry，从而编码动态修改BeanDefinition。例如往BeanDefinition中添加一个新的BeanDefinition。

这两个接口是在AbstractApplicationContext#refresh方法中执行到invokeBeanFactoryPostProcessors(beanFactory);方法时被执行的。

3、示例代码如下：
```java
@Repository
public class OrderDao {

	public void query() {
		System.out.println("OrderDao query...");
	}
}
public class OrderService {

	private OrderDao orderDao;

	public void setDao(OrderDao orderDao) {
		this.orderDao = orderDao;
	}

	public void init() {
		System.out.println("OrderService init...");
	}

	public void query() {
		orderDao.query();
	}
}

@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {

	@Override
	public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
		//向Spring容器中注册OrderService
		BeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition(OrderService.class)
				//这里的属性名是根据setter方法
				.addPropertyReference("dao", "orderDao")
				.setInitMethodName("init")
				.setScope(BeanDefinition.SCOPE_SINGLETON)
				.getBeanDefinition();

		registry.registerBeanDefinition("orderService", beanDefinition);

	}

	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		// 在这里修改orderService bean的scope为PROTOTYPE
		BeanDefinition beanDefinition = beanFactory.getBeanDefinition("orderService");
		beanDefinition.setScope(BeanDefinition.SCOPE_PROTOTYPE);
	}
}
```

回到上面，我们拿`ConfigurationClassPostProcessor`来说：
在Spring中ConfigurationClassPostProcessor同时实现了BeanDefinitionRegistryPostProcessor接口和其父类接口中的方法。
* 1、ConfigurationClassPostProcessor#postProcessBeanFactory：主要负责对Full Configuration 配置进行增强，拦截@Bean方法来确保增强执行@Bean方法的语义。
* 2、ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry：负责扫描我们的程序，根据程序的中Bean创建BeanDefinition，并注册到容器中。

我们进入到：
```java
private void loadBeanDefinitionsForConfigurationClass(
		ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

	if (trackedConditionEvaluator.shouldSkip(configClass)) {
		String beanName = configClass.getBeanName();
		if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
			this.registry.removeBeanDefinition(beanName);
		}
		this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
		return;
	}

	if (configClass.isImported()) {
		registerBeanDefinitionForImportedConfigurationClass(configClass);
	}
	for (BeanMethod beanMethod : configClass.getBeanMethods()) {
		loadBeanDefinitionsForBeanMethod(beanMethod);
	}

	loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
	loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```
其中，我们可以看到：
* 1、通过检查是否有·@import·注解，来注册该导入类到容器中
```java
if (configClass.isImported()) {
		registerBeanDefinitionForImportedConfigurationClass(configClass);
	}
```
* 2、遍历`@Configuration`类中的`@bean`注解，将其类注册到容器中
```java
if (configClass.isImported()) {
		registerBeanDefinitionForImportedConfigurationClass(configClass);
	}
```

#### 4、refresh
这个方法就是正式进行bean的处理的主要逻辑
```java
@Override
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
		prepareRefresh();

		// Tell the subclass to refresh the internal bean factory.
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// Prepare the bean factory for use in this context.
		prepareBeanFactory(beanFactory);

		try {
			// Allows post-processing of the bean factory in context subclasses.
			postProcessBeanFactory(beanFactory);

			// Invoke factory processors registered as beans in the context.
			invokeBeanFactoryPostProcessors(beanFactory);

			// Register bean processors that intercept bean creation.
			registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
			initMessageSource();

			// Initialize event multicaster for this context.
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
			onRefresh();

			// Check for listener beans and register them.
			registerListeners();

			// Instantiate all remaining (non-lazy-init) singletons.
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
			finishRefresh();
		}

		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();

			// Reset 'active' flag.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}

		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			resetCommonCaches();
		}
	}
}
```
前面说的一些扩展点类都是在这里才处理的，spring的扩展机制后面会有专门的文章来讲解。


## SpringMVC
而在web项目中，我们一般都是使用的spring mvc，Spring Framework本身没有Web功能，Spring MVC使用WebApplicationContext类扩展ApplicationContext，使得拥有web功能。那么，Spring MVC是如何在web环境中创建IoC容器呢？web环境中的IoC容器的结构又是什么结构呢？web环境中，Spring IoC容器是怎么启动呢？

#### 1、配置
以Tomcat为例，在Web容器中使用Spirng MVC，必须进行四项的配置：
* 修改web.xml，添加servlet定义；
* 编写servletname-servlet.xml（servletname是在web.xm中配置DispactherServlet时使servlet-name的值）配置；
* contextConfigLocation初始化参数
* 配置ContextLoaderListerner；
  示例配置如下：
```xml
   <!-- servlet定义：前端处理器，接受的HTTP请求和转发请求的类 -->
    <servlet>
        <servlet-name>court</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <!-- court-servlet.xml：定义WebAppliactionContext上下文中的bean -->
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath*:court-servlet.xml</param-value>
        </init-param>
        <load-on-startup>0</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>court</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
    <!-- 配置contextConfigLocation初始化参数：指定Spring IoC容器需要读取的定义了非web层的Bean（DAO/Service）的XML文件路径 -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/court-service.xml</param-value>
    </context-param>
    <!-- 配置ContextLoaderListerner：Spring MVC在Web容器中的启动类，负责Spring IoC容器在Web上下文中的初始化 -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
```
在web.xml配置文件中，有两个主要的配置：ContextLoaderListener和DispatcherServlet。同样的关于spring配置文件的相关配置也有两部分：context-param和DispatcherServlet中的init-param。那么，这两部分的配置有什么区别呢？它们都担任什么样的职责呢？

在Spring MVC中，Spring Context是以父子的继承结构存在的。Web环境中存在一个ROOT Context，这个Context是整个应用的根上下文，是其他context的双亲Context。同时Spring MVC也对应的持有一个独立的Context，它是ROOT Context的子上下文。
对于这样的Context结构在Spring MVC中是如何实现的呢？下面就先从ROOT Context入手，ROOT Context是在ContextLoaderListener中配置的，ContextLoaderListener读取context-param中的contextConfigLocation指定的配置文件，创建ROOT Context。

#### 2、启动过程

Spring MVC启动过程大致分为两个过程：
* ContextLoaderListener初始化，实例化IoC容器，并将此容器实例注册到ServletContext中；
* DispatcherServlet初始化；

tomcat在启动的时候，会依次执行listeners的初始化，也就是执行该ContextLoaderListener的初始化，最终会调用下面的代码:
```java
public void contextInitialized(ServletContextEvent event) {
	this.initWebApplicationContext(event.getServletContext());
}

 
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
     //PS : ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE=WebApplicationContext.class.getName() + ".ROOT" 根上下文的名称
     //PS : 默认情况下，配置文件的位置和名称是： DEFAULT_CONFIG_LOCATION = "/WEB-INF/applicationContext.xml" 
     //在整个web应用中，只能有一个根上下文
     if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
         throw new IllegalStateException("Cannot initialize context because there is already a root application context present - " + "check whether you have multiple ContextLoader* definitions in your web.xml!");
     }
 
     Log logger = LogFactory.getLog(ContextLoader.class);
     servletContext.log("Initializing Spring root WebApplicationContext");
     if (logger.isInfoEnabled()) {
         logger.info("Root WebApplicationContext: initialization started");
     }
     long startTime = System.currentTimeMillis();
 
     try {
         // Store context in local instance variable, to guarantee that
         // it is available on ServletContext shutdown.
         if (this.context == null) {
             // 在这里执行了创建WebApplicationContext的操作
             this.context = createWebApplicationContext(servletContext);
         }
         if (this.context instanceof ConfigurableWebApplicationContext) {
             ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
             if (!cwac.isActive()) {
                 // The context has not yet been refreshed -> provide services such as
                 // setting the parent context, setting the application context id, etc
                 if (cwac.getParent() == null) {
                     // The context instance was injected without an explicit parent ->
                     // determine parent for root web application context, if any.
                     ApplicationContext parent = loadParentContext(servletContext);
                     cwac.setParent(parent);
                 }
                 configureAndRefreshWebApplicationContext(cwac, servletContext);
             }
         }
         // PS: 将根上下文放置在servletContext中
         servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
 
         ClassLoader ccl = Thread.currentThread().getContextClassLoader();
         if (ccl == ContextLoader.class.getClassLoader()) {
             currentContext = this.context;
         } else if (ccl != null) {
             currentContextPerThread.put(ccl, this.context);
         }
 
         if (logger.isDebugEnabled()) {
             logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
         }
         if (logger.isInfoEnabled()) {
             long elapsedTime = System.currentTimeMillis() - startTime;
             logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
         }
 
         return this.context;
     } catch (RuntimeException ex) {
         logger.error("Context initialization failed", ex);
         servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
         throw ex;
     } catch (Error err) {
         logger.error("Context initialization failed", err);
         servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
         throw err;
     }
 }


```
我们注意到这样一句`configureAndRefreshWebApplicationContext(cwac, servletContext);`
这个就是具体创建容器的方法，我们进入去看看
```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
	if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
		// The application context id is still set to its original default value
		// -> assign a more useful id based on available information
		String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
		if (idParam != null) {
			wac.setId(idParam);
		}
		else {
			// Generate default id...
			wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
					ObjectUtils.getDisplayString(sc.getContextPath()));
		}
	}

	wac.setServletContext(sc);
	String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
	if (configLocationParam != null) {
		wac.setConfigLocation(configLocationParam);
	}

	// The wac environment's #initPropertySources will be called in any case when the context
	// is refreshed; do it eagerly here to ensure servlet property sources are in place for
	// use in any post-processing or initialization that occurs below prior to #refresh
	ConfigurableEnvironment env = wac.getEnvironment();
	if (env instanceof ConfigurableWebEnvironment) {
		((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
	}

	customizeContext(sc, wac);
	wac.refresh();
}
```
我们注意到`wac.refresh();`看起来是不是有点熟悉了，进入看看：
```java
public final void refresh() throws BeansException, IllegalStateException {
	try {
		super.refresh();
	}
	catch (RuntimeException ex) {
		WebServer webServer = this.webServer;
		if (webServer != null) {
			webServer.stop();
		}
		throw ex;
	}
}
```
这里的super根据继承关系，我们知道，最终就是进入到了springframework中的refresh中，这个方法我们在上面已经说过了。


## SpringBoot
启动入口方法如下：
```java
public static void main(String[] args) {
        SpringApplication.run(ConsulApplication.class, args);
    }
```

通过代码的层层调用，最终会走到这样的代码中:
```java
public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
        this.configureHeadlessProperty();
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        listeners.starting();

        Collection exceptionReporters;
        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
            this.configureIgnoreBeanInfo(environment);
            Banner printedBanner = this.printBanner(environment);
            context = this.createApplicationContext();
            exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
            this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            this.refreshContext(context);
            this.afterRefresh(context, applicationArguments);
            stopWatch.stop();
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
            }

            listeners.started(context);
            this.callRunners(context, applicationArguments);
        } catch (Throwable var10) {
            this.handleRunFailure(context, var10, exceptionReporters, listeners);
            throw new IllegalStateException(var10);
        }

        try {
            listeners.running(context);
            return context;
        } catch (Throwable var9) {
            this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
            throw new IllegalStateException(var9);
        }
    }
```
可以看到，又走到了大家都熟悉的spring启动代码里面去了。

综上：可以看出，不管是哪种系列的spring，最终都会走到spring基本的启动流程中，无非就是根据自己的特性需要加了一些额外的处理罢了。