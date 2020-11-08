# SpringCloudConsul服务注册原理

## 前言
按照SpringBoot的一贯作法来说这里会有一个starter pom
对于spring starter项目，我们一般首先看该文件：spring.provides
下面我们来看下spring-cloud-starter-consul，进入该meta-inf文件夹下的spring.provides:
```sh
provides: spring-cloud-consul-discovery
```
可以看到该starter会自动引入`spring-cloud-consul-discovery`包。

同样的，对于一个spring项目，我们一般首先查看该文件：spring.factories
下面我们来看下spring-cloud-consul-discovery，进入其meta-inf文件夹下的spring.factories：
```sh
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.consul.discovery.RibbonConsulAutoConfiguration,\
org.springframework.cloud.consul.discovery.configclient.ConsulConfigServerAutoConfiguration,\
org.springframework.cloud.consul.serviceregistry.ConsulAutoServiceRegistrationAutoConfiguration,\
org.springframework.cloud.consul.serviceregistry.ConsulServiceRegistryAutoConfiguration,\
org.springframework.cloud.consul.discovery.ConsulDiscoveryClientConfiguration
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.consul.discovery.configclient.ConsulDiscoveryClientConfigServiceBootstrapConfiguration

```
从上面我们可以看到，服务注册和自动服务注册，配置，服务发现等功能都提供了对应的自动注册的逻辑。
下面我们来看下自动服务注册的逻辑

## 服务自动注册
通过上面我们知道，在容器启动的时候，会执行`ConsulAutoServiceRegistrationAutoConfiguration`的自动配置。
`ConsulAutoServiceRegistrationAutoConfiguration`部分代码如下：
```java
@Bean
@ConditionalOnMissingBean
public ConsulAutoServiceRegistration consulAutoServiceRegistration(ConsulServiceRegistry registry, AutoServiceRegistrationProperties autoServiceRegistrationProperties, ConsulDiscoveryProperties properties, ConsulAutoRegistration consulRegistration) {
	return new ConsulAutoServiceRegistration(registry, autoServiceRegistrationProperties, properties, consulRegistration);
}
```
可以看到，这里通过`@Bean`自动注入了`ConsulAutoServiceRegistration`类
而`ConsulAutoServiceRegistration`又是继承的`AbstractAutoServiceRegistration`抽象类，自动注册的大部分逻辑都在这个类里面，下面就来详细的看看这个类。

我们看下类的继承关系：
```java
public abstract class AbstractAutoServiceRegistration<R extends Registration> implements AutoServiceRegistration, ApplicationContextAware, ApplicationListener<WebServerInitializedEvent> {
```

#### 1、ApplicationContextAware
从上面的继承关系，我们看到，`AbstractAutoServiceRegistration`实现了`ApplicationContextAware`接口。那么这个接口有什么作用呢？

`ApplicationContextAware`的作用是可以方便获取Spring容器ApplicationContext，从而可以获取容器内的Bean。
这里我们来看下`ApplicationContextAware`这个接口：
```java
public interface ApplicationContextAware extends Aware {
    void setApplicationContext(ApplicationContext var1) throws BeansException;
}
```
`ApplicationContextAware`接口只有一个方法，如果实现了这个方法，那么Spring创建这个实现类的时候就会自动执行这个方法，把`ApplicationContext`注入到这个类中，也就是说，spring 在启动的时候就需要实例化这个 class（如果是懒加载就是你需要用到的时候实例化），在实例化这个 class 的时候，发现它包含这个 `ApplicationContextAware` 接口的话，sping 就会调用这个对象的 `setApplicationContext` 方法，把 `applicationContext Set` 进去了。

我们看下`AbstractAutoServiceRegistration`实现的这个方法：
```java
public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
	this.context = applicationContext;
	this.environment = this.context.getEnvironment();
}

protected ApplicationContext getContext() {
        return this.context;
    }
```

#### 2、ApplicationListener
从上面的继承关系，我们看到，`AbstractAutoServiceRegistration`实现了`ApplicationContextAware`接口。那么这个接口又有什么作用呢？

ApplicationContext事件机制是观察者设计模式的实现，通过ApplicationEvent类和ApplicationListener接口，可以实现ApplicationContext事件处理；
如果容器中存在ApplicationListener的Bean，当ApplicationContext调用publishEvent方法发送事件时，对应Bean的`onApplicationEvent`会被触发。
下面为`ApplicationListener`接口代码：
```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
    void onApplicationEvent(E var1);
}
```

同样的，`AbstractAutoServiceRegistration`实现了该方法：
```java
public void onApplicationEvent(WebServerInitializedEvent event) {
        this.bind(event);
    }
```

### 注册过程
从上面知道了，当收到web server初始化成功的事件之后，会触发bind方法，代码如下：
```java
public void bind(WebServerInitializedEvent event) {
	ApplicationContext context = event.getApplicationContext();
	if (!(context instanceof ConfigurableWebServerApplicationContext) || !"management".equals(((ConfigurableWebServerApplicationContext)context).getServerNamespace())) {
		this.port.compareAndSet(0, event.getWebServer().getPort());
		this.start();
	}
}

public void start() {
	if (!this.isEnabled()) {
		if (logger.isDebugEnabled()) {
			logger.debug("Discovery Lifecycle disabled. Not starting");
		}

	} else {
		if (!this.running.get()) {
			this.context.publishEvent(new InstancePreRegisteredEvent(this, this.getRegistration()));
			this.register();
			if (this.shouldRegisterManagement()) {
				this.registerManagement();
			}

			this.context.publishEvent(new InstanceRegisteredEvent(this, this.getConfiguration()));
			this.running.compareAndSet(false, true);
		}

	}
}
```
上面做了几件事：
* 1、更新服务监听端口
* 2、发布预注册事件
* 3、开始注册
* 4、发布注册成功事件
* 5、更新服务运行标志

下面来具体看看注册过程
```java
protected void register() {
        this.serviceRegistry.register(this.getRegistration());
    }
```
调用了serviceRegistry接口：
```java
public interface ServiceRegistry<R extends Registration> {
    void register(R registration);

    void deregister(R registration);

    void close();

    void setStatus(R registration, String status);

    <T> T getStatus(R registration);
}

```
因为我们这里是使用的consul，因此，consul实现了该类
```java
public class ConsulServiceRegistry implements ServiceRegistry<ConsulRegistration> {
    public void register(ConsulRegistration reg) {
        log.info("Registering service with consul: " + reg.getService());

        try {
            this.client.agentServiceRegister(reg.getService(), this.properties.getAclToken());
            NewService service = reg.getService();
            if (this.heartbeatProperties.isEnabled() && this.ttlScheduler != null && service.getCheck() != null && service.getCheck().getTtl() != null) {
                this.ttlScheduler.add(reg.getInstanceId());
            }
        } catch (ConsulException var3) {
            if (this.properties.isFailFast()) {
                log.error("Error registering service with consul: " + reg.getService(), var3);
                ReflectionUtils.rethrowRuntimeException(var3);
            }

            log.warn("Failfast is false. Error registering service with consul: " + reg.getService(), var3);
        }

    }
}
```
上面代码主要做了几件事：
* 1、调用consul-api库方法注册服务信息，最后会简单说下consul-api。
```java
public Response<Void> agentServiceRegister(NewService newService, String token) {
        UrlParameters tokenParam = token != null ? new SingleUrlParameters("token", token) : null;
        String json = GsonFactory.getGson().toJson(newService);
        RawResponse rawResponse = this.rawClient.makePutRequest("/v1/agent/service/register", json, new UrlParameters[]{tokenParam});
        if (rawResponse.getStatusCode() == 200) {
            return new Response((Object)null, rawResponse);
        } else {
            throw new OperationException(rawResponse);
        }
    }
```
很简单，也就是调用consul server提供的接口直接注册而已。

* 2、开启服务周期性健康检查: 根据是否开启了该特性以及是否提供了检查接口等等条件
  下面是具体的调度逻辑：
```java
public void add(String instanceId) {
        ScheduledFuture task = this.scheduler.scheduleAtFixedRate(new TtlScheduler.ConsulHeartbeatTask(instanceId), this.configuration.computeHearbeatInterval().toStandardDuration().getMillis());
        ScheduledFuture previousTask = (ScheduledFuture)this.serviceHeartbeats.put(instanceId, task);
        if (previousTask != null) {
            previousTask.cancel(true);
        }

    }
```

#### 附：Consul-API源码之Client
在consul-api的架构里面，设计了各种不同类型的client，用于封装不同的请求类型。
比如 `AclClient, AgentClient, CatalogClient, CoordinateClient, EventClient, HealthClient, KeyValueClient, QueryClient, SessionClient, StatusClient`

每个client都依赖一个原始的客户端ConsulRawClient，下面来看看这个客户端代码：
除了一些构造函数之外，主要是以下几个方法
```java
public HttpResponse makeGetRequest(String endpoint, List<UrlParameters> urlParams) {
	String url = this.prepareUrl(this.agentAddress + endpoint);
	url = Utils.generateUrl(url, urlParams);
	HttpRequest request = com.ecwid.consul.transport.HttpRequest.Builder.newBuilder().setUrl(url).build();
	return this.httpTransport.makeGetRequest(request);
}

public HttpResponse makePutRequest(String endpoint, String content, UrlParameters... urlParams) {
	String url = this.prepareUrl(this.agentAddress + endpoint);
	url = Utils.generateUrl(url, urlParams);
	HttpRequest request = com.ecwid.consul.transport.HttpRequest.Builder.newBuilder().setUrl(url).setContent(content).build();
	return this.httpTransport.makePutRequest(request);
}

public HttpResponse makeDeleteRequest(Request request) {
	String url = this.prepareUrl(this.agentAddress + request.getEndpoint());
	url = Utils.generateUrl(url, request.getUrlParameters());
	HttpRequest httpRequest = com.ecwid.consul.transport.HttpRequest.Builder.newBuilder().setUrl(url).addHeaders(Utils.createTokenMap(request.getToken())).build();
	return this.httpTransport.makeDeleteRequest(httpRequest);
}
```
可以看出主要还是将url和相关参数交给httpclient库来进行通信和得到response

下面来看看agentConsulClient，示例代码如下：
```java
public Response<Map<String, Check>> getAgentChecks() {
	HttpResponse httpResponse = this.rawClient.makeGetRequest("/v1/agent/checks", new UrlParameters[0]);
	if (httpResponse.getStatusCode() == 200) {
		Map<String, Check> value = (Map)GsonFactory.getGson().fromJson(httpResponse.getContent(), (new TypeToken<Map<String, Check>>() {
		}).getType());
		return new Response(value, httpResponse);
	} else {
		throw new OperationException(httpResponse);
	}
}

public Response<Map<String, Service>> getAgentServices() {
	HttpResponse httpResponse = this.rawClient.makeGetRequest("/v1/agent/services", new UrlParameters[0]);
	if (httpResponse.getStatusCode() == 200) {
		Map<String, Service> agentServices = (Map)GsonFactory.getGson().fromJson(httpResponse.getContent(), (new TypeToken<Map<String, Service>>() {
		}).getType());
		return new Response(agentServices, httpResponse);
	} else {
		throw new OperationException(httpResponse);
	}
}
```
可以看到各个子类客户端主要是提供了consul后端的各个url接口。