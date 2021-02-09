# SpringCloudRibbon负载均衡实现原理

在SpringCloud中，我们最常使用到的负载均衡库就是ribbon。
使用方式，一般是通过自动配置类注入，然后在类中定义负载均衡实例bean
```java
@Configuration
public class RestTemplateConfig {
@Value("${ssl.protocol:TLSv1.2}")
private String sslProtocol;

/**
 * 注入RestTemplate
 *
 * @return RestTemplate Bean
 * @throws KeyManagementException   KeyManagementException
 * @throws NoSuchAlgorithmException NoSuchAlgorithmException
 */
@Bean(name = "frameworkTemplate")
@LoadBalanced
public RestTemplate getRestTemplate() throws KeyManagementException, NoSuchAlgorithmException {
	SSLContext sslContext = SSLContext.getInstance(sslProtocol);
	x509TrustManager(sslContext);
	SSLConnectionSocketFactory csf = new SSLConnectionSocketFactory(sslContext, NoopHostnameVerifier.INSTANCE);
	CloseableHttpClient httpClient = HttpClients.custom().setSSLSocketFactory(csf).build();
	HttpComponentsClientHttpRequestFactory requestFactory = new HttpComponentsClientHttpRequestFactory();

	requestFactory.setHttpClient(httpClient);
	RestTemplate restTemplate = new RestTemplate(requestFactory);
	restTemplate.getMessageConverters().set(1, new StringHttpMessageConverter(StandardCharsets.UTF_8));
	return restTemplate;
}
```
大家注意没有，我们没有直接跟Ribbon打交道，是不是很奇怪，下面就来看看是为什么？

### @LoadBalanced注解
先来看看@LoadBalanced注解，对应类为：
```java
@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {
	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();

	@Autowired(required = false)
	private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();

	@Bean
	public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
			final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
		return () -> restTemplateCustomizers.ifAvailable(customizers -> {
			for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
				for (RestTemplateCustomizer customizer : customizers) {
					customizer.customize(restTemplate);
				}
			}
		});
	}

	@Bean
	@ConditionalOnMissingBean
	public LoadBalancerRequestFactory loadBalancerRequestFactory(
			LoadBalancerClient loadBalancerClient) {
		return new LoadBalancerRequestFactory(loadBalancerClient, this.transformers);
	}

	@Configuration
	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
	static class LoadBalancerInterceptorConfig {

		@Bean
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}

		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
				List<ClientHttpRequestInterceptor> list = new ArrayList<>(
						restTemplate.getInterceptors());
				list.add(loadBalancerInterceptor);
				restTemplate.setInterceptors(list);
			};
		}

	}
}
```
可以看到这里注入了`LoadBalancerInterceptor`这个拦截器和`loadBalancerRequestFactory`这个负载均衡请求工厂类，
记住这两个，在后面讲解RestTemplate时会看到。

上面我们留了一个疑问，为什么我们没有直接使用Ribbon库呢。看到这里，我们注意到：
```java
@Bean
public LoadBalancerInterceptor ribbonInterceptor(
		LoadBalancerClient loadBalancerClient,
		LoadBalancerRequestFactory requestFactory) {
	return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
}
```
这里的拦截器注入了负载均衡客户端，那么这个客户端就是RibbonClient。
```java
public class RibbonLoadBalancerClient implements LoadBalancerClient {}
```

好了，下面我们说说具体的代码实现。
负载均衡的逻辑大部分是在RestTemplate里面，下面我们来具体看看RestTemplate。

### RestTemplate
该接口的继承关系如下：
```java
public class RestTemplate extends InterceptingHttpAccessor implements RestOperations {}
```
1、RestOperations：定义Rest请求的基本接口

2、InterceptingHttpAccessor：实现Rest请求的适配类，包含有两个核心的属性
* List<ClientHttpRequestInterceptor> http请求拦截器集合，作用是实现真正的请求策略，举例：如果RestTemplate要实现负载均衡，那就需要给RestTemplate提供负载均衡的拦截器实现，在这里提供的是LoadBalancerInterceptor
* ClientHttpRequestFactory

在默认调用链中，restTemplate 进行API调用都会调用 doExecute 方法，此方法主要可以进行如下步骤：
* 使用createRequest创建请求，获取响应
* 判断响应是否异常，处理异常
* 将响应消息体封装为java对象

```java
@Nullable
protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,
		@Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {

	Assert.notNull(url, "URI is required");
	Assert.notNull(method, "HttpMethod is required");
	ClientHttpResponse response = null;
	try {
		ClientHttpRequest request = createRequest(url, method);
		if (requestCallback != null) {
			requestCallback.doWithRequest(request);
		}
		response = request.execute();
		handleResponse(url, method, response);
		return (responseExtractor != null ? responseExtractor.extractData(response) : null);
	}
	catch (IOException ex) {
		String resource = url.toString();
		String query = url.getRawQuery();
		resource = (query != null ? resource.substring(0, resource.indexOf('?')) : resource);
		throw new ResourceAccessException("I/O error on " + method.name() +
				" request for \"" + resource + "\": " + ex.getMessage(), ex);
	}
	finally {
		if (response != null) {
			response.close();
		}
	}
}
```

#### 1、创建请求
下面我们来看看它的createRequest(url, method)方法，从上面的继承关系来看，这里的createRequest是执行的`HttpAccessor`这个类
```java
protected ClientHttpRequest createRequest(URI url, HttpMethod method) throws IOException {
	ClientHttpRequest request = getRequestFactory().createRequest(url, method);
	if (logger.isDebugEnabled()) {
		logger.debug("HTTP " + method.name() + " " + url);
	}
	return request;
}
```
通过继承调用关系，我们进入getRequestFactory方法：
```java
@Override
public ClientHttpRequestFactory getRequestFactory() {
	List<ClientHttpRequestInterceptor> interceptors = getInterceptors();
	if (!CollectionUtils.isEmpty(interceptors)) {
		ClientHttpRequestFactory factory = this.interceptingRequestFactory;
		if (factory == null) {
			factory = new InterceptingClientHttpRequestFactory(super.getRequestFactory(), interceptors);
			this.interceptingRequestFactory = factory;
		}
		return factory;
	}
	else {
		return super.getRequestFactory();
	}
}
```
代码的意思很简单：就是如果存在拦截器，那么客户端请求工厂类就是`InterceptingClientHttpRequestFactory`，如果没有则是`SimpleClientHttpRequestFactory`
下面我们主要来看下拦截器场景:
```java
@Override
protected ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod, ClientHttpRequestFactory requestFactory) {
	return new InterceptingClientHttpRequest(requestFactory, this.interceptors, uri, httpMethod);
}

protected InterceptingClientHttpRequest(ClientHttpRequestFactory requestFactory,
		List<ClientHttpRequestInterceptor> interceptors, URI uri, HttpMethod method) {

	this.requestFactory = requestFactory;
	this.interceptors = interceptors;
	this.method = method;
	this.uri = uri;
}
```

#### 2、执行请求
回到最上面的方法代码：
```java
ClientHttpRequest request = createRequest(url, method);
if (requestCallback != null) {
	requestCallback.doWithRequest(request);
}
response = request.execute()
```
创建了请求类之后，开始执行请求，这里的execute是调用的父类，然后父类会调用`executeInternal`方法，而子类`InterceptingClientHttpRequest`重写了此方法：
```java
@Override
protected final ClientHttpResponse executeInternal(HttpHeaders headers, byte[] bufferedOutput) throws IOException {
	InterceptingRequestExecution requestExecution = new InterceptingRequestExecution();
	return requestExecution.execute(this, bufferedOutput);
}

@Override
public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
	if (this.iterator.hasNext()) {
		ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
		return nextInterceptor.intercept(request, body, this);
	}
	else {
		HttpMethod method = request.getMethod();
		Assert.state(method != null, "No standard HTTP method");
		ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
		request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
		if (body.length > 0) {
			if (delegate instanceof StreamingHttpOutputMessage) {
				StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) delegate;
				streamingOutputMessage.setBody(outputStream -> StreamUtils.copy(body, outputStream));
			}
			else {
				StreamUtils.copy(body, delegate.getBody());
			}
		}
		return delegate.execute();
	}
}
```
上面的代码会依次执行请求拦截器方法`intercept`，这里的拦截器自然就是该类`LoadBalancerInterceptor`
```java
@Override
public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
		final ClientHttpRequestExecution execution) throws IOException {
	final URI originalUri = request.getURI();
	String serviceName = originalUri.getHost();
	Assert.state(serviceName != null,
			"Request URI does not contain a valid hostname: " + originalUri);
	return this.loadBalancer.execute(serviceName,
			this.requestFactory.createRequest(request, body, execution));
}
```
这里会先去创建一个负载均衡的请求类，然后调用负载均衡客户端的执行方法，这里的负载均衡客户端类就是`RibbonLoadBalancerClient`:
```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)
			throws IOException {
	ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
	Server server = getServer(loadBalancer, hint);
	if (server == null) {
		throw new IllegalStateException("No instances available for " + serviceId);
	}
	RibbonServer ribbonServer = new RibbonServer(serviceId, server,
			isSecure(server, serviceId),
			serverIntrospector(serviceId).getMetadata(server));

	return execute(serviceId, ribbonServer, request);
}
```

#### 3、选择server
接着上面的代码分析，在上面我们看到`Server server = getServer(loadBalancer, hint);`，我们根据调用链进入`ZoneAwareLoadBalancer`，看看它的初始化：
```java
public class ZoneAwareLoadBalancer<T extends Server> extends DynamicServerListLoadBalancer<T> {}
```
它最终初始化都是调用的父类的方法：
```java
public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping,
                                         ServerList<T> serverList, ServerListFilter<T> filter,
                                         ServerListUpdater serverListUpdater) {
	super(clientConfig, rule, ping);//这里绑定路由规则，ping策略到client，ping的策略会以task的形式定时去刷新服务列表，也就是loadbalance的本地缓存(serverlist)
	this.serverListImpl = serverList;
	this.filter = filter;
	this.serverListUpdater = serverListUpdater;
	if (filter instanceof AbstractServerListFilter) {
		((AbstractServerListFilter) filter).setLoadBalancerStats(getLoadBalancerStats());
	}
	restOfInit(clientConfig);//首次初始化本地服务列表
}
```

##### 3.1 获取服务列表
根据调用链: `restOfInit -> updateListOfServers -> getUpdatedListOfServers`，最终我们进入到具体的实现类，如果以consul作为服务注册和发现，那么这里的实现类就是`ConsulServerList`:
```java
private List<ConsulServer> getServers() {
	if (this.client == null) {
		return Collections.emptyList();
	}
	String tag = getTag(); // null is ok
	Response<List<HealthService>> response = this.client.getHealthServices(
			this.serviceId, tag, this.properties.isQueryPassing(),
			createQueryParamsForClientRequest(), this.properties.getAclToken());
	if (response.getValue() == null || response.getValue().isEmpty()) {
		return Collections.emptyList();
	}
	return transformResponse(response.getValue());
}
```
具体的consul是怎么维持的server列表，这里就不说了，后续会专门写一篇介绍consul的文章。

到目前为止，我们就知道了，ribbon中实现的负载均衡，会依赖服务注册和发现的客户端，通过它们拿到具体可用的服务列表，然后从这些服务列表中，根据负载均衡策略选择一个server进行请求的发送。

##### 3.2 server探测
在拿到server列表之后，ribbon会对每一个server进行ping的探测:
```java
public void forceQuickPing() {
	if (canSkipPing()) {
		return;
	}
	logger.debug("LoadBalancer [{}]:  forceQuickPing invoking", name);
	
	try {
		new Pinger(pingStrategy).runPinger();
	} catch (Exception e) {
		logger.error("LoadBalancer [{}]: Error running forceQuickPing()", name, e);
	}
}
```

##### 3.3 选择server
在所有的server都探测并更新存活server之后，开始根据具体的rule进行选择server进行请求的发送，这里以RoundRobinRule为例：
```java
@Override
public Server chooseServer(Object key) {
	//负载均衡中维护的zone可用实例小于等于1，走父类的选择策略，不会进行zone的选择
	if (!ENABLED.get() || getLoadBalancerStats().getAvailableZones().size() <= 1) {
		logger.debug("Zone aware logic disabled or there is only one zone");
		return super.chooseServer(key);
	}
	Server server = null;
	try {
		LoadBalancerStats lbStats = getLoadBalancerStats();
		//维护的全量zone快照
		Map<String, ZoneSnapshot> zoneSnapshot = ZoneAvoidanceRule.createSnapshot(lbStats);
		logger.debug("Zone snapshots: {}", zoneSnapshot);
		if (triggeringLoad == null) {
			triggeringLoad = DynamicPropertyFactory.getInstance().getDoubleProperty(
					"ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".triggeringLoadPerServerThreshold", 0.2d);
		}

		if (triggeringBlackoutPercentage == null) {
			triggeringBlackoutPercentage = DynamicPropertyFactory.getInstance().getDoubleProperty(
					"ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".avoidZoneWithBlackoutPercetage", 0.99999d);
		}
		//维护的可用的zone
		Set<String> availableZones = ZoneAvoidanceRule.getAvailableZones(zoneSnapshot, triggeringLoad.get(), triggeringBlackoutPercentage.get());
		logger.debug("Available zones: {}", availableZones);
		if (availableZones != null &&  availableZones.size() < zoneSnapshot.keySet().size()) {
			//随机选择一个zone
			String zone = ZoneAvoidanceRule.randomChooseZone(zoneSnapshot, availableZones);
			logger.debug("Zone chosen: {}", zone);
			if (zone != null) {
				//真正的取server的逻辑是在loadBalancer所绑定的rule，即ZoneAvoidanceRule
				BaseLoadBalancer zoneLoadBalancer = getLoadBalancer(zone);
				server = zoneLoadBalancer.chooseServer(key);
			}
		}
	} catch (Exception e) {
		logger.error("Error choosing server using zone aware logic for load balancer={}", name, e);
	}
	if (server != null) {
		return server;
	} else {
		logger.debug("Zone avoidance logic is not invoked.");
		return super.chooseServer(key);
	}
}
	
public Server choose(ILoadBalancer lb, Object key) {
	if (lb == null) {
		log.warn("no load balancer");
		return null;
	}

	Server server = null;
	int count = 0;
	while (server == null && count++ < 10) {
		List<Server> reachableServers = lb.getReachableServers();
		List<Server> allServers = lb.getAllServers();
		int upCount = reachableServers.size();
		int serverCount = allServers.size();

		if ((upCount == 0) || (serverCount == 0)) {
			log.warn("No up servers available from load balancer: " + lb);
			return null;
		}

		int nextServerIndex = incrementAndGetModulo(serverCount);
		server = allServers.get(nextServerIndex);

		if (server == null) {
			/* Transient. */
			Thread.yield();
			continue;
		}

		if (server.isAlive() && (server.isReadyToServe())) {
			return (server);
		}

		// Next.
		server = null;
	}

	if (count >= 10) {
		log.warn("No available alive servers after 10 tries from load balancer: "
				+ lb);
	}
	return server;
}
```