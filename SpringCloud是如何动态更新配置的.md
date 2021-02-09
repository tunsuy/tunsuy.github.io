# SpringCloud是如何动态更新配置的
spring cloud在config配置管理的基础上，提供了consul config的配置管理和动态监听，那么这里面到底是怎样实现的，本文将为你揭秘。
注：这里讲的动态配置更新不只局限于consul，对于任意的配置都是这样的逻辑，本文将其spring源码进行详细的剖析。

### 前言
对于单体应用架构来说，会使用配置文件管理我们的配置，这就是之前项目中的application.properties或application.yml。如果需要在多环境下使用，传统的做法是复制这些文件命名为application-xxx.properties，并且在启动时配置spring.profiles.active={profile}来指定环境。
在微服务架构下我们可能会有很多的微服务，所以要求的不只是在各自微服务中进行配置，我们需要将所有的配置放在统一平台上进行操作，不同的环境进行不同的配置，运行期间动态调整参数等等。
于是Spring Cloud为我们提供了一个统一的配置管理，那就是`Spring Cloud Config`。

##### spring cloud config简介
它为分布式系统外部配置提供了服务器端和客户端的支持，它包括config server端和 config client端两部分
* Config server端是一个可以横向扩展、集中式的配置服务器，它用于集中管理应用程序各个环境下的配置，默认 使用Git存储配置内容
* Config client 是config server的客户端，用于操作存储在server中的配置属性

### 启动加载扩展点
spring boot提供在 META-INF/spring.factories 文件中增加配置，来实现一些程序中预定义的扩展点。通过这种方式配置的扩展点好处是不局限于某一种接口的实现，而是同一类别的实现。我们查看 spring-cloud-context 包中的 spring.factories 文件，如下所示：
```java
# AutoConfiguration
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration,\
org.springframework.cloud.autoconfigure.LifecycleMvcEndpointAutoConfiguration,\
org.springframework.cloud.autoconfigure.RefreshAutoConfiguration,\
org.springframework.cloud.autoconfigure.RefreshEndpointAutoConfiguration,\
org.springframework.cloud.autoconfigure.WritableEnvironmentEndpointAutoConfiguration
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.cloud.bootstrap.BootstrapApplicationListener,\
org.springframework.cloud.bootstrap.LoggingSystemShutdownListener,\
org.springframework.cloud.context.restart.RestartListener
# Bootstrap components
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration,\
org.springframework.cloud.bootstrap.encrypt.EncryptionBootstrapConfiguration,\
org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration,\
org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration,\
org.springframework.cloud.util.random.CachedRandomPropertySourceAutoConfiguration

```
可以看到`BootstrapConfiguration`下面有一个item，`PropertySourceBootstrapConfiguration`，进入其代码，查看即继承关系，发现其实现了 `ApplicationContextInitializer` 接口，其目的就是在应用程序上下文初始化的时候做一些额外的操作。
在 Bootstrap 阶段，会通过 Spring Ioc 的整个生命周期来初始化所有通过key为`org.springframework.cloud.bootstrap.BootstrapConfiguration` 在 spring.factories 中配置的 Bean。初始化的过程中，会获取所有 `ApplicationContextInitializer` 类型的 Bean，并设置回SpringApplication主流程当中。通过在 SpringApplication 的主流程中回调这些 `ApplicationContextInitializer` 的实例，做一些初始化的操作，即调用`initialize`方法。

下面我们就来看看`PropertySourceBootstrapConfiguration`这个方法：
```java
@Override
public void initialize(ConfigurableApplicationContext applicationContext) {
CompositePropertySource composite = new CompositePropertySource(
		BOOTSTRAP_PROPERTY_SOURCE_NAME);
AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
boolean empty = true;
ConfigurableEnvironment environment = applicationContext.getEnvironment();
for (PropertySourceLocator locator : this.propertySourceLocators) {
	PropertySource<?> source = null;
	//回调所有实现PropertySourceLocator接口实例的locate方法，
	source = locator.locate(environment);
	if (source == null) {
		continue;
	}

	composite.addPropertySource(source);
	empty = false;
}
if (!empty) {
//从当前Enviroment中获取 propertySources
	MutablePropertySources propertySources = environment.getPropertySources();
	//省略...
	//将composite中的PropertySource添加到当前应用上下文的propertySources中
	insertPropertySources(propertySources, composite);
	//省略...
}
```
在这个方法中会回调所有实现 PropertySourceLocator 接口实例的locate方法，
locate 方法返回一个 PropertySource 的实例，统一add到CompositePropertySource实例中。如果 composite 中有新加的PropertySource,最后将composite中的PropertySource添加到当前应用上下文的propertySources中。

### SpringCloudConsul的配置加载
正如上面说的，在 Bootstrap 阶段，会通过 Spring Ioc 的整个生命周期来初始化所有通过key为`org.springframework.cloud.bootstrap.BootstrapConfiguration` 在 spring.factories 中配置的 Bean。
同样的在spring.factories文件中：
```java
# Auto Configuration
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.consul.config.ConsulConfigAutoConfiguration
# Bootstrap Configuration
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.consul.config.ConsulConfigBootstrapConfiguration
```
我们确实看到了又这样一个key存在，对应value为`ConsulConfigBootstrapConfiguration`类，我们看看该类的实现：
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnConsulEnabled
public class ConsulConfigBootstrapConfiguration {

	@Configuration(proxyBeanMethods = false)
	@EnableConfigurationProperties
	@Import(ConsulAutoConfiguration.class)
	@ConditionalOnProperty(name = "spring.cloud.consul.config.enabled",
			matchIfMissing = true)
	protected static class ConsulPropertySourceConfiguration {

		@Autowired
		private ConsulClient consul;

		@Bean
		@ConditionalOnMissingBean
		public ConsulConfigProperties consulConfigProperties() {
			return new ConsulConfigProperties();
		}

		@Bean
		public ConsulPropertySourceLocator consulPropertySourceLocator(
				ConsulConfigProperties consulConfigProperties) {
			return new ConsulPropertySourceLocator(this.consul, consulConfigProperties);
		}

	}

}
```
我们看到，这里只是注入了一些bean，我们注意下`ConsulPropertySourceLocator`这个类。


正如上面说的，SpringCloudConfig在启动的时候会回调所有实现 `PropertySourceLocator` 接口实例的locate方法，consul就是实现了`PropertySourceLocator`接口，具体类为`ConsulPropertySourceLocator`，实现了`locate`方法：
```java
@Override
@Retryable(interceptor = "consulRetryInterceptor")
public PropertySource<?> locate(Environment environment) {
	if (environment instanceof ConfigurableEnvironment) {
		ConfigurableEnvironment env = (ConfigurableEnvironment) environment;

		String appName = this.properties.getName();

		if (appName == null) {
			appName = env.getProperty("spring.application.name");
		}

		List<String> profiles = Arrays.asList(env.getActiveProfiles());

		String prefix = this.properties.getPrefix();

		List<String> suffixes = new ArrayList<>();
		// 不是文件类型的时候，后缀为 /，否则就是配置文件的后缀
		if (this.properties.getFormat() != FILES) {
			suffixes.add("/");
		} else {
			suffixes.add(".yml");
			suffixes.add(".yaml");
			suffixes.add(".properties");
		}

		// 路径
		String defaultContext = getContext(prefix, this.properties.getDefaultContext());

		for (String suffix : suffixes) {
			this.contexts.add(defaultContext + suffix);
		}
		// 追加环境及文件类型
		for (String suffix : suffixes) {
			addProfiles(this.contexts, defaultContext, profiles, suffix);
		}

		String baseContext = getContext(prefix, appName);

		// 应用名称前缀
		for (String suffix : suffixes) {
			this.contexts.add(baseContext + suffix);
		}
		for (String suffix : suffixes) {
			addProfiles(this.contexts, baseContext, profiles, suffix);
		}

		Collections.reverse(this.contexts);

		CompositePropertySource composite = new CompositePropertySource("consul");

		for (String propertySourceContext : this.contexts) {
			try {
				ConsulPropertySource propertySource = null;
				if (this.properties.getFormat() == FILES) {
					// 获取值
					Response<GetValue> response = this.consul.getKVValue(propertySourceContext, this.properties.getAclToken());
					// 添加当前索引
					addIndex(propertySourceContext, response.getConsulIndex());
					// 如果值不为空，则更新值并初始化
					if (response.getValue() != null) {
						ConsulFilesPropertySource filesPropertySource = new ConsulFilesPropertySource(propertySourceContext, this.consul, this.properties);
						// 解析配置内容
						filesPropertySource.init(response.getValue());
						propertySource = filesPropertySource;
					}
				} else {
					propertySource = create(propertySourceContext, this.contextIndex);
				}
				if (propertySource != null) {
					composite.addPropertySource(propertySource);
				}
			} catch (Exception e) {
				if (this.properties.isFailFast()) {
					log.error("Fail fast is set and there was an error reading configuration from consul.");
					ReflectionUtils.rethrowRuntimeException(e);
				} else {
					log.warn("Unable to load consul config from " + propertySourceContext, e);
				}
			}
		}

		return composite;
	}
	return null;
}
```
获取配置时，根据应用名称，路径，环境及配置类型拼接相应的路径，然后调用 Consul 获取 KV 值的接口，获取相应的配置，根据类型解析后放入环境中


### 配置动态刷新
感知到外部化配置的变更这部分代码的操作是需要用户来完成的。Spring Cloud Config 只提供了具备外部化配置可动态刷新的能力，并不具备自动感知外部化配置发生变更的能力。比如如果你的配置是基于Mysql来实现的，那么在代码里面肯定要有能力感知到配置发生变化了，然后再显示的调用 ContextRefresher 的 refresh方法，从而完成外部化配置的动态刷新(只会刷新使用RefreshScope注解的Bean)。

下面我们来看看config框架是怎么进行动态刷新的？
主要类是这个`ContextRefresher`，刷新方法如下：
```java
public synchronized Set refresh() {

    Map<String, Object> before = extract(
            this.context.getEnvironment().getPropertySources());
    //1、加载最新的值，并替换Envrioment中旧值
    addConfigFilesToEnvironment();
    Set<String> keys = changes(before,
            extract(this.context.getEnvironment().getPropertySources())).keySet();
    this.context.publishEvent(new EnvironmentChangeEvent(context, keys));
    //2、将refresh scope中的Bean 缓存失效: 清空
    this.scope.refreshAll();
    return keys;
}
```
上面ContextRefresher的refresh的方法主要做了两件事：
* 1、触发PropertySourceLocator的locator方法，需要加载最新的值，并替换 Environment 中旧值
* 2、Bean中的引用配置值需要重新注入一遍。重新注入的流程是在Bean初始化时做的操作，那也就是需要将refresh scope中的Bean 缓存失效，当再次从refresh scope中获取这个Bean时，发现取不到，就会重新触发一次Bean的初始化过程。

可以看到上面代码中有这样一句`this.scope.refreshAll()`，其中的scope就是RefreshScope。是用来存放scope类型为refresh类型的Bean（即使用RefreshScope注解标识的Bean），也就是说当一个Bean既不是singleton也不是prototype时，就会从自定义的Scope中去获取(Spring 允许自定义Scope)，然后调用Scope的get方法来获取一个实例，Spring Cloud 正是扩展了Scope，从而控制了整个 Bean 的生命周期。当配置需要动态刷新的时候， 调用this.scope.refreshAll()这个方法，就会将整个RefreshScope的缓存清空，完成配置可动态刷新的可能。

###### 注：关于`ContextRefresh`和`RefreshScope`的初始化配置是在`RefreshAutoConfiguration`类中完成的。而`RefreshAutoConfiguration`类初始化的入口是在spring-cloud-context中的META-INF/spring.factories中配置的。从而完成整个和动态刷新相关的Bean的初始化操作。


### SpringCloudConsul的配置刷新
Consul 监听配置是通过定时任务实现的，涉及的类为`ConfigWatch`
```java
public class ConfigWatch implements ApplicationEventPublisherAware, SmartLifecycle {}
```
该类的初始化是在 `org.springframework.cloud.consul.config.ConsulConfigAutoConfiguration` 中实现的：
```java
@Bean
@ConditionalOnProperty(name = "spring.cloud.consul.config.watch.enabled", matchIfMissing = true)
public ConfigWatch configWatch(ConsulConfigProperties properties,
                               ConsulPropertySourceLocator locator,
                               ConsulClient consul,
                               @Qualifier(CONFIG_WATCH_TASK_SCHEDULER_NAME) TaskScheduler taskScheduler) {
	return new ConfigWatch(properties, consul, locator.getContextIndexes(), taskScheduler);
}
```

我们看到`ConfigWatch` 类实现了 `ApplicationEventPublisherAware` 和 `SmartLifecycle` 接口.
当应用启动后，会调用 其实现的`SmartLifecycle` 的 `start` 方法，然后初始化配置监听，通过向线程池添加一个定时任务，实现配置的定时拉取，定时任务默认周期是 1s
```java
@Override
public void start() {
	if (this.running.compareAndSet(false, true)) {
		this.watchFuture = this.taskScheduler.scheduleWithFixedDelay(
			this::watchConfigKeyValues, this.properties.getWatch().getDelay());
	}
}
```

#### 1、发布事件
定时任务的监听逻辑如下：
```java
// Timed 是 Prometheus 的监控
@Timed("consul.watch-config-keys")
public void watchConfigKeyValues() {
	if (this.running.get()) {
		// 遍历所有的配置的 key
		for (String context : this.consulIndexes.keySet()) {

			// turn the context into a Consul folder path (unless our config format
			// are FILES)
			if (this.properties.getFormat() != FILES && !context.endsWith("/")) {
				context = context + "/";
			}

			// 根据配置返回的 index 判断是否发生变化
			try {
				Long currentIndex = this.consulIndexes.get(context);
				if (currentIndex == null) {
					currentIndex = -1L;
				}

				log.trace("watching consul for context '" + context + "' with index " + currentIndex);

				// use the consul ACL token if found
				String aclToken = this.properties.getAclToken();
				if (StringUtils.isEmpty(aclToken)) {
					aclToken = null;
				}

				// 获取指定的 key
				Response<List<GetValue>> response = this.consul.getKVValues(context, aclToken, new QueryParams(this.properties.getWatch().getWaitTime(), currentIndex));

				// if response.value == null, response was a 404, otherwise it was a
				// 200
				// reducing churn if there wasn't anything
				if (response.getValue() != null && !response.getValue().isEmpty()) {
					Long newIndex = response.getConsulIndex();

					// 判断 key 的 index 是否相等，如果发生变化，则发出 RefreshEvent 事件
					if (newIndex != null && !newIndex.equals(currentIndex)) {
						// don't publish the same index again, don't publish the first
						// time (-1) so index can be primed
						// 没有发布过这个 index 的事件，且不是第一次发布
						if (!this.consulIndexes.containsValue(newIndex) && !currentIndex.equals(-1L)) {
							log.trace("Context " + context + " has new index " + newIndex);
							// 发送事件
							RefreshEventData data = new RefreshEventData(context, currentIndex, newIndex);
							this.publisher.publishEvent(new RefreshEvent(this, data, data.toString()));
						} else if (log.isTraceEnabled()) {
							log.trace("Event for index already published for context " + context);
						}
						this.consulIndexes.put(context, newIndex);
					} else if (log.isTraceEnabled()) {
						log.trace("Same index for context " + context);
					}
				} else if (log.isTraceEnabled()) {
					log.trace("No value for context " + context);
				}

			} catch (Exception e) {
				// only fail fast on the initial query, otherwise just log the error
				if (this.firstTime && this.properties.isFailFast()) {
					log.error("Fail fast is set and there was an error reading configuration from consul.");
					ReflectionUtils.rethrowRuntimeException(e);
				} else if (log.isTraceEnabled()) {
					log.trace("Error querying consul Key/Values for context '" + context + "'", e);
				} else if (log.isWarnEnabled()) {
					// simplified one line log message in the event of an agent
					// failure
					log.warn("Error querying consul Key/Values for context '" + context + "'. Message: " + e.getMessage());
				}
			}
		}
	}
	this.firstTime = false;
}
```
监听时会遍历所有的key，根据 key 从 Consul 获取相应的数据，判断 Index 是否发生变化，如果发生变化，则发送 RefreshEvent 事件，需要手动实现事件监听以响应配置变化。
至于spring是怎样发布事件，监听者又是怎样接收到的，这里面的细节后续有时间再详细剖析。

#### 2、事件监听
现在我们主要来看下`RefreshEvent`发出去之后，监听者的逻辑。

通过函数调用栈，我们找到了这样一个监听者`RefreshEventListener`：
```java
@Override
public void onApplicationEvent(ApplicationEvent event) {
	if (event instanceof ApplicationReadyEvent) {
		handle((ApplicationReadyEvent) event);
	}
	else if (event instanceof RefreshEvent) {
		handle((RefreshEvent) event);
	}
}
```
我们知道，在spring中，监听者都需要实现这样一个方法`onApplicationEvent`，该方法中我们发现有这样一个分支
```java
else if (event instanceof RefreshEvent) {
	handle((RefreshEvent) event);
}
```
这个事件就是上面发出来的，因此这里能够监听到，然后执行回调方法`handle`
```java
public void handle(RefreshEvent event) {
	if (this.ready.get()) { // don't handle events before app is ready
		log.debug("Event received " + event.getEventDesc());
		Set<String> keys = this.refresh.refresh();
		log.info("Refresh keys changed: " + keys);
	}
}

public synchronized Set<String> refresh() {
	Set<String> keys = refreshEnvironment();
	this.scope.refreshAll();
	return keys;
}
```

主要的逻辑在`refreshEnvironment`方法中：
```java
public synchronized Set<String> refreshEnvironment() {
	Map<String, Object> before = extract(
			this.context.getEnvironment().getPropertySources());
	addConfigFilesToEnvironment();
	Set<String> keys = changes(before,
			extract(this.context.getEnvironment().getPropertySources())).keySet();
	this.context.publishEvent(new EnvironmentChangeEvent(this.context, keys));
	return keys;
}
```
我们知道`this.context.getEnvironment().getPropertySources()`是获取env中的所有配置源，然后将其交给extract进行处理。
extract方法就是将配置源中的各种格式的配置，比如map、yml、properties类型等等，统一转换为map类型，这样就可以通过统一key-value形式获取到任意想要的配置值。
上面这段代码的主要逻辑就是：
* 1、获取所有的旧的（更新之前的）配置值
* 2、重新通过应用初始方式更新所有的配置值`addConfigFilesToEnvironment`
* 3、将最新的值跟旧的值进行对比，找出所有的更新过的key
* 4、重新发布配置变更时间`EnvironmentChangeEvent`，将更新过的key传递给该事件

#### 3、Env配置更新
下面来说下第二点：重新通过应用初始方式更新所有的配置值`addConfigFilesToEnvironment`，
```java
/* For testing. */ ConfigurableApplicationContext addConfigFilesToEnvironment() {
	ConfigurableApplicationContext capture = null;
	try {
		StandardEnvironment environment = copyEnvironment(
				this.context.getEnvironment());
		SpringApplicationBuilder builder = new SpringApplicationBuilder(Empty.class)
				.bannerMode(Mode.OFF).web(WebApplicationType.NONE)
				.environment(environment);
		// Just the listeners that affect the environment (e.g. excluding logging
		// listener because it has side effects)
		builder.application()
				.setListeners(Arrays.asList(new BootstrapApplicationListener(),
						new ConfigFileApplicationListener()));
		capture = builder.run();
		if (environment.getPropertySources().contains(REFRESH_ARGS_PROPERTY_SOURCE)) {
			environment.getPropertySources().remove(REFRESH_ARGS_PROPERTY_SOURCE);
		}
		MutablePropertySources target = this.context.getEnvironment()
				.getPropertySources();
		String targetName = null;
		for (PropertySource<?> source : environment.getPropertySources()) {
			String name = source.getName();
			if (target.contains(name)) {
				targetName = name;
			}
			if (!this.standardSources.contains(name)) {
				if (target.contains(name)) {
					target.replace(name, source);
				}
				else {
					if (targetName != null) {
						target.addAfter(targetName, source);
						// update targetName to preserve ordering
						targetName = name;
					}
					else {
						// targetName was null so we are at the start of the list
						target.addFirst(source);
						targetName = name;
					}
				}
			}
		}
	}
    ......
|
```
我们看到有这样一句`SpringApplicationBuilder`，这里它是生成了一个spring应用对象的生成器，然后执行它的run方法，
也就是说，这里会重新执行一遍spring的启动流程，所有的启动初始类都会重新执行，包括上面提到的`ConsulPropertySourceLocator`类的`locate`方法，这里就会再次向consul server发起请求获取最新的配置数据，写入env中。
因此后面通过`this.context.getEnvironment().getPropertySources()`得到的就是最新的配置源了。
同时业务中也可以通过` context.getEnvironment().getProperty(key)`拿到任意key的最新值了。

### 刷新scope域
在上面的refresh方法中，我们还剩下这样一句没有讲解：
```java
this.scope.refreshAll();
```
这里主要就是刷新spring容器该scope类型下的所有bean，就可以通过@RefreshScope的bean实例的get方法获取到最新的值了。前提条件是我们需要监听这个事件`RefreshScopeRefreshedEvent`：
```java
@EventListener(classes = RefreshScopeRefreshedEvent.class)
public void updateChange(RefreshScopeRefreshedEvent event) {
	//这里获取到的新的值
	String pwd = redisProperties.getPassword();
	System.out.print("new pwd: " + pwd);
}
```

上面的`EnvironmentChangeEvent`这个事件发生时，@RefreshScope的bean实例还是老的bean，在这个事件里拿到的还是老的值：
```java
@EventListener(classes = EnvironmentChangeEvent.class)
public void updateChange(EnvironmentChangeEvent event) {
	Set<String> updatedKeys = event.getKeys();
	System.out.print(updatedKeys);
	for (String key : updatedKeys) {
		if (key.equals("redis.password")) {
			System.out.print("new password: " + context.getEnvironment().getProperty(key));
			//    do something
		}
	}
	//这里获取到的还是旧的值
	String pwd = redisProperties.getPassword();
	System.out.print("old pwd: " + pwd);
}
```

具体是怎样刷新scope域的，后面有时间再专门讲解。

##### 参考链接： https://juejin.im/post/6845166890461954062