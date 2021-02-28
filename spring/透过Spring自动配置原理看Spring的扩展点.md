# 透过Spring自动配置原理看Spring的扩展点

### EnableAutoConfiguration注解
spring的自动配置就是得益于这个`EnableAutoConfiguration`注解：
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	/**
	 * Exclude specific auto-configuration classes such that they will never be applied.
	 * @return the classes to exclude
	 */
	Class<?>[] exclude() default {};

	/**
	 * Exclude specific auto-configuration class names such that they will never be
	 * applied.
	 * @return the class names to exclude
	 * @since 1.3.0
	 */
	String[] excludeName() default {};

}
```
该注解导入了一个类`AutoConfigurationImportSelector`，从该类的名字我们可以看到，它是一个selector类，那么就具有selector类的特性。

我们都知道，spring启动的时候会自动执行selector类的方法`selectImports`方法，从而将配置类自动注入容器中。下面就来看看是在哪里执行的这个方法

### 启动加载selector
spring的启动流程在之前的文章中已经讲过了，我们知道启动的时候会经过`AbstractApplicationContext`类的`refresh`方法
在该方法中有这样一句：
```java
// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
	PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

	// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
	// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
	if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}
}
				
```
最终会进入到`ConfigurationClassPostProcessor`类的`processConfigBeanDefinitions`方法：
```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
		List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
		String[] candidateNames = registry.getBeanDefinitionNames();

		

	// Parse each @Configuration class
	ConfigurationClassParser parser = new ConfigurationClassParser(
			this.metadataReaderFactory, this.problemReporter, this.environment,
			this.resourceLoader, this.componentScanBeanNameGenerator, registry);

	Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
	Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
	do {
		parser.parse(candidates);
		parser.validate();

}
```
这里就是在处理标有Configuration注解的类，我们进入`parser.parse(candidates);`方法：
```java
ConfigurationClassParser.java

private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
		Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter,
		boolean checkForCircularImports) {

	if (importCandidates.isEmpty()) {
		return;
	}

	if (checkForCircularImports && isChainedImportOnStack(configClass)) {
		this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
	}
	else {
		this.importStack.push(configClass);
		try {
			for (SourceClass candidate : importCandidates) {
				if (candidate.isAssignable(ImportSelector.class)) {
					// Candidate class is an ImportSelector -> delegate to it to determine imports
					Class<?> candidateClass = candidate.loadClass();
					ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
							this.environment, this.resourceLoader, this.registry);
					Predicate<String> selectorFilter = selector.getExclusionFilter();
					if (selectorFilter != null) {
						exclusionFilter = exclusionFilter.or(selectorFilter);
					}
					if (selector instanceof DeferredImportSelector) {
						this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
					}
					else {
						String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
						Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames, exclusionFilter);
						processImports(configClass, currentSourceClass, importSourceClasses, exclusionFilter, false);
					}
				}
				else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
					// Candidate class is an ImportBeanDefinitionRegistrar ->
					// delegate to it to register additional bean definitions
					Class<?> candidateClass = candidate.loadClass();
					ImportBeanDefinitionRegistrar registrar =
							ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
									this.environment, this.resourceLoader, this.registry);
					configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
				}
				else {
					// Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
					// process it as an @Configuration class
					this.importStack.registerImport(
							currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
					processConfigurationClass(candidate.asConfigClass(configClass), exclusionFilter);
				}
			}
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(
					"Failed to process import candidates for configuration class [" +
					configClass.getMetadata().getClassName() + "]", ex);
		}
		finally {
			this.importStack.pop();
		}
	}
}
```
我们看到有这样一句`selector.selectImports(currentSourceClass.getMetadata())`。
看到这里，是不是就很清楚了，spring就是这样在启动的时候实现了自动导入配置类的。

下面我们再来看看自动配置类的内部逻辑
### AutoConfigurationImportSelector
我们进入selector类，找到`selectImports`方法：
```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return NO_IMPORTS;
	}
	AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
	return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}

protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return EMPTY_ENTRY;
	}
	AnnotationAttributes attributes = getAttributes(annotationMetadata);
	List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
	configurations = removeDuplicates(configurations);
	Set<String> exclusions = getExclusions(annotationMetadata, attributes);
	checkExcludedClasses(configurations, exclusions);
	configurations.removeAll(exclusions);
	configurations = getConfigurationClassFilter().filter(configurations);
	fireAutoConfigurationImportEvents(configurations, exclusions);
	return new AutoConfigurationEntry(configurations, exclusions);
}
```
这里我们注意到这行代码`List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);`，这句代码就是获取所有jar中meta-inf/spring.factories文件中的类，下面具体来看看：
```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
	List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
			getBeanClassLoader());
	Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
			+ "are using a custom packaging, make sure that file is correct.");
	return configurations;
}

public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
	String factoryTypeName = factoryType.getName();
	return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
}

private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
	MultiValueMap<String, String> result = cache.get(classLoader);
	if (result != null) {
		return result;
	}

	try {
		Enumeration<URL> urls = (classLoader != null ?
				classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
				ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
		result = new LinkedMultiValueMap<>();
		while (urls.hasMoreElements()) {
			URL url = urls.nextElement();
			UrlResource resource = new UrlResource(url);
			Properties properties = PropertiesLoaderUtils.loadProperties(resource);
			for (Map.Entry<?, ?> entry : properties.entrySet()) {
				String factoryTypeName = ((String) entry.getKey()).trim();
				for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
					result.add(factoryTypeName, factoryImplementationName.trim());
				}
			}
		}
		cache.put(classLoader, result);
		return result;
	}
	catch (IOException ex) {
		throw new IllegalArgumentException("Unable to load factories from location [" +
				FACTORIES_RESOURCE_LOCATION + "]", ex);
	}
}
```
上面的代码中，我们注意到这样一个常量`public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";`，意思就是获取该文件中定义的类名，并作为列表返回。
我们看一下这个文件的示例：
```yml
# AutoConfiguration
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.client.CommonsClientAutoConfiguration,\
org.springframework.cloud.client.ReactiveCommonsClientAutoConfiguration,\
org.springframework.cloud.client.discovery.composite.CompositeDiscoveryClientAutoConfiguration,\
org.springframework.cloud.client.discovery.composite.reactive.ReactiveCompositeDiscoveryClientAutoConfiguration,\
org.springframework.cloud.client.discovery.noop.NoopDiscoveryClientAutoConfiguration,\
org.springframework.cloud.client.discovery.simple.SimpleDiscoveryClientAutoConfiguration,\
org.springframework.cloud.client.discovery.simple.reactive.SimpleReactiveDiscoveryClientAutoConfiguration,\
org.springframework.cloud.client.hypermedia.CloudHypermediaAutoConfiguration,\
org.springframework.cloud.client.loadbalancer.AsyncLoadBalancerAutoConfiguration,\
org.springframework.cloud.client.loadbalancer.LoadBalancerAutoConfiguration,\
org.springframework.cloud.client.loadbalancer.reactive.LoadBalancerBeanPostProcessorAutoConfiguration,\
org.springframework.cloud.client.loadbalancer.reactive.ReactorLoadBalancerClientAutoConfiguration,\
org.springframework.cloud.client.loadbalancer.reactive.ReactiveLoadBalancerAutoConfiguration,\
org.springframework.cloud.client.serviceregistry.ServiceRegistryAutoConfiguration,\
org.springframework.cloud.commons.httpclient.HttpClientConfiguration,\
org.springframework.cloud.commons.util.UtilAutoConfiguration,\
org.springframework.cloud.configuration.CompatibilityVerifierAutoConfiguration,\
org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationAutoConfiguration
# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.cloud.client.HostInfoEnvironmentPostProcessor
# Failure Analyzers
org.springframework.boot.diagnostics.FailureAnalyzer=\
org.springframework.cloud.configuration.CompatibilityNotMetFailureAnalyzer

```

好了，我们回到之前的`getAutoConfigurationEntry`方法，
```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return EMPTY_ENTRY;
	}
	AnnotationAttributes attributes = getAttributes(annotationMetadata);
	List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
	configurations = removeDuplicates(configurations);
	Set<String> exclusions = getExclusions(annotationMetadata, attributes);
	checkExcludedClasses(configurations, exclusions);
	configurations.removeAll(exclusions);
	configurations = getConfigurationClassFilter().filter(configurations);
	fireAutoConfigurationImportEvents(configurations, exclusions);
	return new AutoConfigurationEntry(configurations, exclusions);
}
```
上面的代码很清晰：
1、在配置文件中拿到类名列表
2、对该列表去重
3、根据注解属性，去掉指定排除的类
4、根据自定义`AutoConfigurationImportFilter`类过滤
5、回调`AutoConfigurationImportListener`监听的导入事件


### AutoConfigurationImportFilter
在上面我们知道的，加载配置类的时候会根据条件过滤某些类，那么这里就来看下具体的实现:
```java
configurations = getConfigurationClassFilter().filter(configurations);

private ConfigurationClassFilter getConfigurationClassFilter() {
	if (this.configurationClassFilter == null) {
		List<AutoConfigurationImportFilter> filters = getAutoConfigurationImportFilters();
		for (AutoConfigurationImportFilter filter : filters) {
			invokeAwareMethods(filter);
		}
		this.configurationClassFilter = new ConfigurationClassFilter(this.beanClassLoader, filters);
	}
	return this.configurationClassFilter;
}
```
从上面的代码中，我们看到，这里的listener是怎么来的呢？
```java
protected List<AutoConfigurationImportFilter> getAutoConfigurationImportFilters() {
	return SpringFactoriesLoader.loadFactories(AutoConfigurationImportFilter.class, this.beanClassLoader);
}
```
看到这个代码里面的调用，又回到了刚刚讲解获取自动配置类的地方，只是这里传入的`Class<T> factoryType`变成了`AutoConfigurationImportFilter`，总结一下，也就是说：
filter也是从`META-INF/spring.factories`配置文件里面获取出来的，示例如下：
```yml
# Auto Configuration Import Filters
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
org.springframework.boot.autoconfigure.condition.OnBeanCondition,\
org.springframework.boot.autoconfigure.condition.OnClassCondition,\
org.springframework.boot.autoconfigure.condition.OnWebApplicationCondition
```

##### 扩展点一
看到这里我们也可以总结出spring其中的一个扩展点：
如果业务中需要根据条件动态的控制某个类的导入，则可以实现`AutoConfigurationImportFilter`接口，并且实现方法`match`，然后放入META-INF/spring.factories配置文件中Key为：org.springframework.context.ApplicationContextInitializer的value中。spring在导入配置类的时候，会自动执行这个扩展方法。
注：spring另外封装了一个类`ConfigurationClassFilter`来统一处理

### AutoConfigurationImportListener
在上面我们知道的，加载配置类的时候会回调`AutoConfigurationImportListener`监听的导入事件，那么这里就来看下具体的实现:
```java
private void fireAutoConfigurationImportEvents(List<String> configurations, Set<String> exclusions) {
	List<AutoConfigurationImportListener> listeners = getAutoConfigurationImportListeners();
	if (!listeners.isEmpty()) {
		AutoConfigurationImportEvent event = new AutoConfigurationImportEvent(this, configurations, exclusions);
		for (AutoConfigurationImportListener listener : listeners) {
			invokeAwareMethods(listener);
			listener.onAutoConfigurationImportEvent(event);
		}
	}
}
```
从上面的代码中，我们看到，这里的listener是怎么来的呢？
```java
protected List<AutoConfigurationImportListener> getAutoConfigurationImportListeners() {
	return SpringFactoriesLoader.loadFactories(AutoConfigurationImportListener.class, this.beanClassLoader);
}
```
看到这个代码里面的调用，又回到了刚刚讲解获取自动配置类的地方，只是这里传入的`Class<T> factoryType`变成了`AutoConfigurationImportListener`，总结一下，也就是说：
listener也是从`META-INF/spring.factories`配置文件里面获取出来的，示例如下：
```yml
# Auto Configuration Import Listeners
org.springframework.boot.autoconfigure.AutoConfigurationImportListener=\
org.springframework.boot.autoconfigure.condition.ConditionEvaluationReportAutoConfigurationImportListener
```

##### 扩展点二
看到这里我们也可以总结出spring其中的一个扩展点：
如果业务中需要监控某个类的导入，则可以实现`AutoConfigurationImportListener`接口，并且实现方法`onAutoConfigurationImportEvent`，然后放入META-INF/spring.factories配置文件中Key为：org.springframework.context.ApplicationContextInitializer的value中。spring在导入配置类的时候，会自动执行这个扩展方法。


### spring.factories
从上面我们知道了，spring就是通过`spring.factories`文件让外部实现自定义扩展的，其实，该文件还可以定义其他一些classtype的key，比如在springboot的启动中，有如下代码：
```java
public static void main(String[] args) {
	SpringApplication.run(xxx.class, args);
}
	
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
	this.resourceLoader = resourceLoader;
	Assert.notNull(primarySources, "PrimarySources must not be null");
	this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
	this.webApplicationType = WebApplicationType.deduceFromClasspath();
	setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
	this.mainApplicationClass = deduceMainApplicationClass();
}
```
可以看出，springboot在启动的时候会加载spring.factories中`ApplicationContextInitializer`和`ApplicationListener`为key的类

看到这里我们也可以总结出spring的一个扩展点：
##### 扩展点三
如果业务需要在启动的时候自定义下容器的初始化动作，则可以实现`ApplicationContextInitializer`接口，并且实现方法`initialize`，然后放入META-INF/spring.factories配置文件中Key为：org.springframework.context.ApplicationContextInitializer的value中。


看到这里，我们是不是可以想象下，我们是不是可以在spring.factories中自定义key-value，然后通过库方法自行解析，达到任意扩展的效果呢？答案是可以的。