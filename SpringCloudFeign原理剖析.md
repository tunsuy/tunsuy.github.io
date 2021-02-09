## SpringCloudFeign原理剖析

##### Feign是什么？

简单来说，feign是用在微服务中，各个微服务间的调用。它是通过声明式的方式来定义接口，而不用实现接口。接口的实现由它通过spring bean的动态注册来实现的。
feign让服务间的调用变得简单，不用各个服务去处理http client相关的逻辑。并且它里面集成了ribbon用来做负载均衡，通过集成了hystrix用来做服务熔断和降级。

在feign的使用中，我们主要用到它的两个注解，下面一一来说明。

### 注解
#### 1、@EnableFeignClients
* 用于表示该客户端开启Feign方式调用
* 创建一个关于FeignClient的工厂Bean，这个工厂Bean会通过@FeignClient收集调用信息（服务、接口等），创建动态代理对象

#### 2、@FeignClient
负责标识一个用于业务调用的Client，给FactoryBean提供创建代理对象，提供基础数据（类名、方法、服务名、URI等），作用是提供这些静态配置


### 实现原理
一般对于一个spring boot服务来说，如果要使用feign，都会这样定义:
```java
@Configuration
@EnableScheduling
@EnableDiscoveryClient
@EnableFeignClients(value = {"com.ts"})
@MapperScan(value = {"com.ts.xx.xx"}, nameGenerator = FullBeanNameGenerator.class)
public class xxxxAutoConfig {
}
```
这里就使用到了上面说的`EnableFeignClients`这个注解，这个注解有什么用呢？我们进入该注解：
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {}
```
可以看到该注解导入了`FeignClientsRegistrar`类，我们进入其中：
```java
class FeignClientsRegistrar
		implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {}
	@Override
	public void registerBeanDefinitions(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
		registerDefaultConfiguration(metadata, registry);
		registerFeignClients(metadata, registry);
	}		

```
整个过程大概就是，通过配置类，或者package路径做扫描，收集FeignClient的静态信息，每个Client会把他的基本信息，类名、方法、服务名等绑定到FactoryBean上，这样就就具备了生成一个动态代理类的基本条件。

这里穿插2个知识点
#### 1、spring bean的动态注册
在spring中有两类bean：
* 普通的bean：通过xml配置或者注解配置
* 工厂bean：也是一个Bean，这个Bean我们业务中不会直接用到，它主要是用于生成其他的一些Bean，内部进行了高度封装，非常容易实现配置化的管理，屏蔽了实现细节。
  动态注册是解决什么问题，根据客户端的配置动态的，也就是可以按需做bean的注册。

##### 1.1 使用方式
实现`ImportBeanDefinitionRegistrar`接口，重点实现registerBeanDefinitions方法，该接口需要配合`@Configuration`和`@Import`注解，`@Configuration`定义Java格式的Spring配置文件，`@Import`注解导入实现了`ImportBeanDefinitionRegistrar`接口的类。
这也就是上面我们看到的`FeignClientsRegistrar`。那feign为什么要这样做呢？因为需要生成不同的代理类的实现bean。

##### 1.2 实现原理
所有实现了该接口的类的都会被`ConfigurationClassPostProcessor`处理，`ConfigurationClassPostProcessor`实现了`BeanFactoryPostProcessor`接口，所以`ImportBeanDefinitionRegistrar`中动态注册的bean是优先于依赖它的bean初始化的，也能被aop、validator等机制处理。

#### 2、FactoryBean
FactoryBean的特殊之处在于它可以向容器中注册两个Bean，一个是它本身，一个是FactoryBean.getObject()方法返回值所代表的Bean。

我们知道：在Spring容器启动阶段，会调用到`refresh()`方法，在`refresh()`中有调用了`finishBeanFactoryInitialization()`方法，最终会调用到`beanFactory.preInstantiateSingletons()`方法。
在`getObjectForBeanInstance()`方法中会先判断bean是不是FactoryBean，如果不是，就直接返回Bean。如果是FactoryBean，且name是以`&`符号开头，那么表示的是获取FactoryBean的原生对象，也会直接返回。如果name不是以&符号开头，那么表示要获取FactoryBean中`getObject()`方法返回的对象。会先尝试从`FactoryBeanRegistrySupport`类的`factoryBeanObjectCache`这个缓存map中获取，如果缓存中存在，则返回，如果不存在，则去调用`getObjectFromFactoryBean()`方法。`getObjectFromFactoryBean()`方法中，主要是通过调用`doGetObjectFromFactoryBean()`方法得到bean，然后对bean进行处理，最后放入缓存。而且还会针对单例bean和非单例bean做区分处理，对于单例bean，会在创建完后，将其放入到缓存中，非单例bean则不会放入缓存，而是每次都会重新创建。


#### 注册FeignClient
我们回到`FeignClientsRegistrar`类中，下面接着看下注册FeignClient的实现，代码如下：
```java
public void registerFeignClients(AnnotationMetadata metadata,
            BeanDefinitionRegistry registry) {
        //创建一个类扫描器
        ClassPathScanningCandidateComponentProvider scanner = getScanner();
        scanner.setResourceLoader(this.resourceLoader);

        Set<String> basePackages;
        //获取EnableFeignClients注解包含的属性
        Map<String, Object> attrs = metadata
                .getAnnotationAttributes(EnableFeignClients.class.getName());
        //这是一个FeignClient注解过滤器
        AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
                FeignClient.class);
        final Class<?>[] clients = attrs == null ? null
                : (Class<?>[]) attrs.get("clients");
        //准备出待扫描的路径
        if (clients == null || clients.length == 0) {
            scanner.addIncludeFilter(annotationTypeFilter);
            basePackages = getBasePackages(metadata);
        }
        else {
            final Set<String> clientClasses = new HashSet<>();
            basePackages = new HashSet<>();
            for (Class<?> clazz : clients) {
                basePackages.add(ClassUtils.getPackageName(clazz));
                clientClasses.add(clazz.getCanonicalName());
            }
            AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
                @Override
                protected boolean match(ClassMetadata metadata) {
                    String cleaned = metadata.getClassName().replaceAll("\\$", ".");
                    return clientClasses.contains(cleaned);
                }
            };
            scanner.addIncludeFilter(
                    new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
        }

        for (String basePackage : basePackages) {
        //先扫描出含有@FeignClient注解的类
            Set<BeanDefinition> candidateComponents = scanner
                    .findCandidateComponents(basePackage);
            for (BeanDefinition candidateComponent : candidateComponents) {
                if (candidateComponent instanceof AnnotatedBeanDefinition) {
                    // verify annotated class is an interface
                    AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
                    AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
                    //该类必须是接口，做强制校验
                    Assert.isTrue(annotationMetadata.isInterface(),
                            "@FeignClient can only be specified on an interface");
                    //获取该类的所有属性
                    Map<String, Object> attributes = annotationMetadata
                            .getAnnotationAttributes(
                                    FeignClient.class.getCanonicalName());
                    //获取服务名称
                    String name = getClientName(attributes);
                    registerClientConfiguration(registry, name,
                            attributes.get("configuration"));
                    //循环对使用@FeignClient注解的不同的类，做工厂bean注册
                    registerFeignClient(registry, annotationMetadata, attributes);
                }
            }
        }
    }

    private void registerFeignClient(BeanDefinitionRegistry registry,
            AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
        //含有@FeignClient该类的名称
        String className = annotationMetadata.getClassName();
        //创建一个BeanDefinitionBuilder，内含AbstractBeanDefinition，指定待创建的Bean的名字是FeignClientFactoryBean
        BeanDefinitionBuilder definition = BeanDefinitionBuilder
                .genericBeanDefinition(FeignClientFactoryBean.class);
        validate(attributes);
        definition.addPropertyValue("url", getUrl(attributes));
        definition.addPropertyValue("path", getPath(attributes));
        String name = getName(attributes);
        definition.addPropertyValue("name", name);
        definition.addPropertyValue("type", className);
        definition.addPropertyValue("decode404", attributes.get("decode404"));
        definition.addPropertyValue("fallback", attributes.get("fallback"));
        definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

        String alias = name + "FeignClient";
        AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

        boolean primary = (Boolean)attributes.get("primary"); // has a default, won't be null

        beanDefinition.setPrimary(primary);

        String qualifier = getQualifier(attributes);
        if (StringUtils.hasText(qualifier)) {
            alias = qualifier;
        }

        BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
                new String[] { alias });
        //动态注册
        BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
    }
    //BeanDefinitionReaderUtils.registerBeanDefinition
    public static void registerBeanDefinition(
            BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
            throws BeanDefinitionStoreException {

        // Register bean definition under primary name.
        String beanName = definitionHolder.getBeanName();
        //BeanDefinitionRegistry.registerBeanDefinition实现具体的注册,会告知他需要注册的类名、以及AbstractBeanDefinition
        registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

        // Register aliases for bean name, if any.
        String[] aliases = definitionHolder.getAliases();
        if (aliases != null) {
            for (String alias : aliases) {
                registry.registerAlias(beanName, alias);
            }
        }
    }
```

#### 创建代理类
在上面提到的`FactoryBean`，可以看到`FeignClientFactoryBean`继承了它，通过调用实现类的getObject完成代理类的创建：
```java
@Override
  public Object getObject() throws Exception {
    return getTarget();
  }
  <T> T getTarget() {
    FeignContext context = applicationContext.getBean(FeignContext.class);
    Feign.Builder builder = feign(context);
    //先校验基础属性，基础属性是在FeignClientsRegistrar中给动态Bean，添加属性addPropertyValue时候赋值的
    //URL为空，则使用http://serviceName的方式拼接
    if (!StringUtils.hasText(this.url)) {
      String url;
      if (!this.name.startsWith("http")) {
        url = "http://" + this.name;
      }
      else {
        url = this.name;
      }
      url += cleanPath();
      //创建动态的代理对象，采用JDK动态代理
      return (T) loadBalance(builder, context, new HardCodedTarget<>(this.type,
          this.name, url));
    }
    //含有url，也就是FeignClient的注解接口里是一个绝对地址
    if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
      this.url = "http://" + this.url;
    }
    String url = this.url + cleanPath();
    Client client = getOptional(context, Client.class);
    if (client != null) {
      if (client instanceof LoadBalancerFeignClient) {
        // not load balancing because we have a url,
        // but ribbon is on the classpath, so unwrap
        client = ((LoadBalancerFeignClient)client).getDelegate();
      }
      builder.client(client);
    }
    Targeter targeter = get(context, Targeter.class);
    return (T) targeter.target(this, builder, context, new HardCodedTarget<>(
        this.type, this.name, url));
  }
```

从上面代码可以知道，如果用户在FeignClient的注解中直接使用了URL，这种方式一般用于调试环境，直接指定一个服务的绝对地址，这种情况下不会走负载均衡，走默认的Client,代码如下：
```java
@Override
    public Response execute(Request request, Options options) throws IOException {
      HttpURLConnection connection = convertAndSend(request, options);
      return convertResponse(connection).toBuilder().request(request).build();
    }
```
如果用户在FeignClient中使用了seriveName，那么请求地址将会是http://serviceName，这种情况下是需要走负载均衡的，通过如下代码发现Feign的负载均衡也是基于Ribbon实现：
```java
public Response execute(Request request, Request.Options options) throws IOException {
		try {
			URI asUri = URI.create(request.url());
			String clientName = asUri.getHost();
			URI uriWithoutHost = cleanUrl(request.url(), clientName);
			FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
					this.delegate, request, uriWithoutHost);

			IClientConfig requestConfig = getClientConfig(options, clientName);
			return lbClient(clientName).executeWithLoadBalancer(ribbonRequest,
					requestConfig).toResponse();
		}
		catch (ClientException e) {
			IOException io = findIOException(e);
			if (io != null) {
				throw io;
			}
			throw new RuntimeException(e);
		}
	}
```


### 相关配置
#### 1、服务配置
大多数情况下，我们对于服务调用的超时时间可能会根据实际服务的特性做 一 些调整，所以仅仅依靠默认的全局配置是不行的。 在使用SpringCloud Feign的时候，针对各个服务客户端进行个性化配置的方式与使用SpringCloud Ribbon时的配置方式是 一 样的， 都采用<client>. ribbon.key=value 的格式进行 设置。在定义Feign客户端的时候， 我们使用了@FeignClient注解。 在初始化过程中，SpringCloud Feign会根据该注解的name属性或value属性指定的服务名， 自动创建一 个同名的Ribbon客户端。也就是说，在之前的示例中，使用@FeignClient(value= "cloud-provider")来创 建 Feign 客 户 端 的 时 候 ， 同时也创建了一个 名为cloud-provider的Ribbon客户端。 既然如此， 我们就可以使用@FeignClient注解中的name或value属性值来设置对应的Ribbon参数， 比如：
```sh
cloud-provider.ribbon.ConnectTimeout = 500 //请求连接的超时时间。
cloud-provider.ribbon.ReadTimeout = 2000  //请求处理的超时时间。
cloud-provider.ribbon.OkToRetryOnAllOperations = true //对所有操作请求都进行重试。
cloud-provider.ribbon.MaxAutoRetriesNextServer = 2 //切换实例的重试次数。
cloud-provider.ribbon.MaxAutoRetries = 1 //对当前实例的重试次数。
```

#### 2、日志配置
Spring Cloud Feign 在构建被 @FeignClient 注解修饰的服务客户端时，会为每 一 个客户端都创建 一 个 feign.Logger 实例，我们可以利用该日志对象的 DEBUG 模式来帮助分析 Feign 的请求细节。可以在 application.properties 文件中使用 logging.level.<FeignClient> 的参数配置格式来开启指定 Feign 客户端的 DEBUG 日志， 其中<FeignClient> 为 Feign 客户端定义接口的完整路径， 比如针对本文中我们实现的 HelloService 可以按如下配置开启：
```sh
logging.level.com.wuzz.demo.HelloService = DEBUG
```
但是， 只是添加了如上配置， 还无法实现对 DEBUG 日志的输出。 这时由于 Feign 客户端默认的 Logger.Level 对象定义为 NONE 级别， 该级别不会记录任何 Feign 调用过程中的信息， 所以我们需要调整它的级别， 针对全局的日志级别， 可以在应用主类中直接加入 Logger.Level 的 Bean 创建， 具体如下：
```java
@Bean
Logger.Level feignLoggerLevel() {
    return Logger.Level.FULL;
}
```
或者添加个配置类：
```java
@Configuration
public class FullLogConfiguration {
@Bean
Logger.Level feignLoggerLevel() {
    return Logger.Level.FULL;
}
@FeignClient(name = "cloud-provider", configuration = FullLogConfiguration.class)
public interface TestService {
    
}
```

#### 3、服务降级
根据目标接口，创建一个实现了FallbackFactory的类 
```java
@Component
public class HystrixClientService implements FallbackFactory<ClientService> {
    @Override
    public ClientService create(Throwable throwable) {
        return new ClientService() {
            @Override
            public String test() {
                return "test sdfsdfsf";
            }
        };
    }
}
```
在目标接口上的@FeignClient中添加fallbackFactory属性值
```java
@FeignClient(value ="cloud-provider", fallbackFactory = HystrixClientService.class)
public interface ClientService {

    @RequestMapping(value ="/test",method= RequestMethod.GET)
    String test() ;
}
```
修改 application.yml ,添加一下
```xml
feign:
   hystrix:
    enabled: true
```