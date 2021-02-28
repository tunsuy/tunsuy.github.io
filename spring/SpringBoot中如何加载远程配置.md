# SpringBoot中如何加载远程配置

本文章将通过结合consul config来讲解在springboot中如何加载远程配置：通过consul config加载consul server中存储的配置。

我们先来说下在spring中常规的加载配置文件的方式。

### 加载配置文件方式

对于一个工程来说，我们一般都会需要有各种配置，在spring工程里面，一般都是yml或者properties文件，如下所示：
```yaml
server:
  port: 9991 # 端口

spring:
  application:
    name: ts-service1 # 应用名称
  profiles:
    active: dev # 指定环境，默认加载 default 环境
  cloud:
    consul:
      # Consul 服务器地址
      host: 51.6.196.200
      port: 8500
      # 配置中心相关配置
      config:
        # 是否启用配置中心，默认值 true 开启
        enabled: true
        # 设置配置的基本文件夹，默认值 config 可以理解为配置文件所在的最外层文件夹
        prefix: config
        # 设置应用的文件夹名称，默认值 application 一般建议设置为微服务应用名称
        default-context: tsService
        # 配置环境分隔符，默认值 "," 和 default-context 配置项搭配
        # 例如应用 orderService 分别有环境 default、dev、test、prod
        # 只需在 config 文件夹下创建 orderService、orderService-dev、orderService-test、orderService-prod 文件夹即可
        profile-separator: '-'
        # 指定配置格式为 yaml
        format: YAML
        # Consul 的 Key/Values 中的 Key，Value 对应整个配置文件
        data-key: redisConfig
        # 以上配置可以理解为：加载 config/orderService/ 文件夹下 Key 为 orderServiceConfig 的 Value 对应的配置信息
        watch:
          # 是否开启自动刷新，默认值 true 开启
          enabled: true
          # 刷新频率，单位：毫秒，默认值 1000
          delay: 1000
      # 服务发现相关配置
      discovery:
        register: true                                # 是否需要注册
        instance-id: ${spring.application.name}-01    # 注册实例 id（必须唯一）
        service-name: ${spring.application.name}      # 服务名称
        port: ${server.port}                          # 服务端口
        prefer-ip-address: true                       # 是否使用 ip 地址注册
        ip-address: ${spring.cloud.client.ip-address} # 服务请求 ip
```

那么对于读取这些配置文件中的值，一般有如下几种方式
* 1、使用 @Value("${property}") 读取比较简单的配置信息。
* 2、通过@ConfigurationProperties读取并通过@Component与 bean 绑定
* 3、通过@ConfigurationProperties读取并在使用的地方使用@EnableConfigurationProperties注册需要的配置 bean
* 4、通过@PropertySource读取指定 properties 文件

注：spring加载配置文件有个默认的加载顺序的，根据存放的路径来决定。

## 拉取远程配置
我们知道，上面说的那些一般要求配置都必须是本地，而且格式只能是 properties(或者 yaml)。那么，如果我们有远程配置，如何把他引入进来来呢。
主要有以下三步：
* 1、编写PropertySource：编写一个类继承EnumerablePropertySource，然后实现它的抽象方法即可。
* 2、编写PropertySourceLocator： PropertySourceLocator 其实就是用来定位我们前面的PropertySource，需要重写的方法只有一个，就是返回一个PropertySource对象。
* 3、配置PropertySourceLocator生效

下面就以consul为例，剖析下它是怎么做的

consul的配置示例如下：
```yml
spring:
  application:
    name: ts-service1 # 应用名称
  profiles:
    active: dev # 指定环境，默认加载 default 环境
  cloud:
    consul:
      # Consul 服务器地址
      host: 51.6.196.200
      port: 8500
      # 配置中心相关配置
      config:
        # 是否启用配置中心，默认值 true 开启
        enabled: true
        # 设置配置的基本文件夹，默认值 config 可以理解为配置文件所在的最外层文件夹
        prefix: config
        # 设置应用的文件夹名称，默认值 application 一般建议设置为微服务应用名称
        default-context: tsService
        # 配置环境分隔符，默认值 "," 和 default-context 配置项搭配
        # 例如应用 orderService 分别有环境 default、dev、test、prod
        # 只需在 config 文件夹下创建 orderService、orderService-dev、orderService-test、orderService-prod 文件夹即可
        profile-separator: '-'
        # 指定配置格式为 yaml
        format: YAML
        # Consul 的 Key/Values 中的 Key，Value 对应整个配置文件
        data-key: redisConfig
        # 以上配置可以理解为：加载 config/orderService/ 文件夹下 Key 为 orderServiceConfig 的 Value 对应的配置信息
        watch:
          # 是否开启自动刷新，默认值 true 开启
          enabled: true
          # 刷新频率，单位：毫秒，默认值 1000
          delay: 1000
```

一般对于配置文件，都有一个对应的配置属性类，consul也不例外：
```java
@ConfigurationProperties("spring.cloud.consul.config")
@Validated
public class ConsulConfigProperties {

	private boolean enabled = true;

	private String prefix = "config";

	@NotEmpty
	private String defaultContext = "application";

	@NotEmpty
	private String profileSeparator = ",";

	@NotNull
	private Format format = Format.KEY_VALUE;

	/**
	 * If format is Format.PROPERTIES or Format.YAML then the following field is used as
	 * key to look up consul for configuration.
	 */
	@NotEmpty
	private String dataKey = "data";

	@Value("${consul.token:${CONSUL_TOKEN:${spring.cloud.consul.token:${SPRING_CLOUD_CONSUL_TOKEN:}}}}")
	private String aclToken;

	private Watch watch = new Watch();

	...
}
```
上面代码对应的就是yml文件中`spring.consul.config`下面的这一部分的配置。

### 1、编写PropertySource
consul config下面有这样一个类：`ConsulPropertySource`，看下它的继承关系：
```java
public class ConsulPropertySource extends EnumerablePropertySource<ConsulClient> {
	private final Map<String, Object> properties = new LinkedHashMap<>();
}
```
可以看到，它使用了一个map来说存储配置数据。

主要看下以下三个方法：
```java
@Override
public Object getProperty(String name) {
	return this.properties.get(name);
}

@Override
public String[] getPropertyNames() {
	Set<String> strings = this.properties.keySet();
	return strings.toArray(new String[strings.size()]);
}
```
这两个方法就是需要继承实现的父类方法，主要就是获取配置信息，那这些配置信息是哪里来的呢？下面看第三个方法：
```java
public void init() {
	if (!this.context.endsWith("/")) {
		this.context = this.context + "/";
	}

	Response<List<GetValue>> response = this.source.getKVValues(this.context,
			this.configProperties.getAclToken(), QueryParams.DEFAULT);

	this.initialIndex = response.getConsulIndex();

	final List<GetValue> values = response.getValue();
	ConsulConfigProperties.Format format = this.configProperties.getFormat();
	switch (format) {
	case KEY_VALUE:
		parsePropertiesInKeyValueFormat(values);
		break;
	case PROPERTIES:
	case YAML:
		parsePropertiesWithNonKeyValueFormat(values, format);
	}
}
```
这里的this.context可以理解为consul中key-value存储里面的key。
this.source就是ConsulClient。

上面代码的逻辑就是：
* 通过consulclient向consul server发起请求，查询前缀key为this.context的value信息
* 根据consul配置文件（也就是工程里面的yml或者properties文件）里面配置的format配置来决定解析该response
* 如果format是key-value，则表示consule server中该this.context对应的value是一个key-value格式的值，按照key-value进行解析放入this.properties中
* 如果format是yml或者properties，则表示consule server中该this.context对应的value是一个yml或者properties格式的值，按照相应的格式进行解析放入this.properties中

注：consul config还提供了另外一个propertySource的实现：
```java
public class ConsulFilesPropertySource extends ConsulPropertySource {
	public void init(GetValue value) {
		if (this.getContext().endsWith(".yml") || this.getContext().endsWith(".yaml")) {
			parseValue(value, YAML);
		}
		else if (this.getContext().endsWith(".properties")) {
			parseValue(value, PROPERTIES);
		}
		else {
			throw new IllegalStateException(
					"Unknown files extension for context " + this.getContext());
		}
	}

}
```
该类继承自上面说的那个类，实现了init方法：主要就是用于直接将获取到的value根据需要解析成yml或者properties格式的数据。

### 2、编写PropertySourceLocator
consul config的实现类如下：
```java
@Order(0)
public class ConsulPropertySourceLocator implements PropertySourceLocator {}
```

如上面所说，我们主要关注下locate方法：
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
		if (this.properties.getFormat() != FILES) {
			suffixes.add("/");
		}
		else {
			suffixes.add(".yml");
			suffixes.add(".yaml");
			suffixes.add(".properties");
		}

		String defaultContext = getContext(prefix,
				this.properties.getDefaultContext());

		for (String suffix : suffixes) {
			this.contexts.add(defaultContext + suffix);
		}
		for (String suffix : suffixes) {
			addProfiles(this.contexts, defaultContext, profiles, suffix);
		}

		String baseContext = getContext(prefix, appName);

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
					Response<GetValue> response = this.consul.getKVValue(
							propertySourceContext, this.properties.getAclToken());
					addIndex(propertySourceContext, response.getConsulIndex());
					if (response.getValue() != null) {
						ConsulFilesPropertySource filesPropertySource = new ConsulFilesPropertySource(
								propertySourceContext, this.consul, this.properties);
						filesPropertySource.init(response.getValue());
						propertySource = filesPropertySource;
					}
				}
				else {
					propertySource = create(propertySourceContext, this.contextIndex);
				}
				if (propertySource != null) {
					composite.addPropertySource(propertySource);
				}
			}
			catch (Exception e) {
				if (this.properties.isFailFast()) {
					log.error(
							"Fail fast is set and there was an error reading configuration from consul.");
					ReflectionUtils.rethrowRuntimeException(e);
				}
				else {
					log.warn("Unable to load consul config from "
							+ propertySourceContext, e);
				}
			}
		}

		return composite;
	}
	return null;
}
```
上面代码的逻辑就是：
* 通过上面配置文件中prefix和default-context、prefix和application.name通过分隔符组合成consule中的key
* 对每一个key，创建`ConsulPropertySource`实例并初始化（上一节我们已经分析过了），将该实例保存下来

### 3、配置启动加载
consul config实现这样一个类：
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
然后在 `META-INF/spring.factories`中配置如下：
```java
# Auto Configuration
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.consul.config.ConsulConfigAutoConfiguration
# Bootstrap Configuration
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.consul.config.ConsulConfigBootstrapConfiguration

```
就是给Spring Boot说，这个是一个启动配置类，spring boot在启动的时候会自动加载。

## 放入environment
上面讲解了对应接口的实现，那么consul的这些实现类是在哪里调用的呢？过程是这样的：
spring boot工程在启动的时候，会执行BootStrapConfiguration的initize方法，`PropertySourceBootstrapConfiguration`的该方法如下：
```java
@Override
public void initialize(ConfigurableApplicationContext applicationContext) {
	List<PropertySource<?>> composite = new ArrayList<>();
	AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
	boolean empty = true;
	ConfigurableEnvironment environment = applicationContext.getEnvironment();
	for (PropertySourceLocator locator : this.propertySourceLocators) {
		Collection<PropertySource<?>> source = locator.locateCollection(environment);
		if (source == null || source.size() == 0) {
			continue;
		}
		List<PropertySource<?>> sourceList = new ArrayList<>();
		for (PropertySource<?> p : source) {
			if (p instanceof EnumerablePropertySource) {
				EnumerablePropertySource<?> enumerable = (EnumerablePropertySource<?>) p;
				sourceList.add(new BootstrapPropertySource<>(enumerable));
			}
			else {
				sourceList.add(new SimpleBootstrapPropertySource(p));
			}
		}
		logger.info("Located property source: " + sourceList);
		composite.addAll(sourceList);
		empty = false;
	}
	if (!empty) {
		MutablePropertySources propertySources = environment.getPropertySources();
		String logConfig = environment.resolvePlaceholders("${logging.config:}");
		LogFile logFile = LogFile.get(environment);
		for (PropertySource<?> p : environment.getPropertySources()) {
			if (p.getName().startsWith(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
				propertySources.remove(p.getName());
			}
		}
		insertPropertySources(propertySources, composite);
		reinitializeLoggingSystem(environment, logConfig, logFile);
		setLogLevels(applicationContext, environment);
		handleIncludedProfiles(environment);
	}
}
```
可以看到，这里代码中，调用了每个`PropertySourceLocator`的实例方法`locateCollection`，该方法里面调用了`locate`方法，也就是回到了上一节所说的内容了。
最后将所有的source放入了`environment`中： `insertPropertySources(propertySources, composite);`

## 读取propertySource
通过上面的方式加载了远程配置之后，我们在其他地方就可以任意读取了，方式如下：
```java
//可以获取整个系统所有的配置：通过这个可以获取到更新之前的数据
ConfigurableEnvironment environment = (ConfigurableEnvironment)context.getEnvironment();
PropertySources sources = environment.getPropertySources();
```