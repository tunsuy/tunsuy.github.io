# 基于Feign的扩展机制实现TLS通信

### 改造spring应用为tls模式
我们在使用springboot运行一个应用的时候，默认是http模式的，但是在生产环境中，一般都要求是https模式
具体做法如下：
1、生成证书（这里只是示例，生产环境需要严格通过CA签发）
```sh
keytool -genkeypair -alias ts_https -keypass ts123 -keyalg RSA -keysize 1024 -validity 365 -keystore d:/ts/ts_https.keystore -storepass ts123
```
根据提示填入相应信息即可

2、spring参数配置
在应用配置文件application.properties中增加如下参数：
```sh
#开启https，配置跟证书一一对应
server.ssl.enabled=true
#指定证书
server.ssl.key-store=classpath:ts_https.keystore
server.ssl.key-store-type=JKS
#别名
server.ssl.key-alias=ts_https
#密码
server.ssl.key-password=ts1234
server.ssl.key-store-password=ts1234
#是否强制认证客户端
server.ssl.client-auth=need
```
对于spring的参数文件，我们一般都可以在IDE中点击该参数，直接就可以跳转到相应的代码实现中，从而知道所有的参数情况，
上面对应的代码文件为：`org\springframework\boot\spring-boot\2.2.4.RELEASE\spring-boot-2.2.4.RELEASE-sources.jar!\org\springframework\boot\web\server\Ssl.java`

注：
如果是需要强制开启双向认证，则需要加上`server.ssl.client-auth=need`配置

大家可能已经注意到了，上面配置的密码是明文，这在实际生产环境中是不允许的，需要密码存储。那是不是直接改成密文，spring就能自动识别呢？我们可以试下，就会发现启动报错：
```sh
org.springframework.boot.web.server.WebServerException: Unable to start embedded Tomcat server
	at org.springframework.boot.web.embedded.tomcat.TomcatWebServer.start(TomcatWebServer.java:215) ~[spring-boot-2.2.4.RELEASE.jar:2.2.4.RELEASE]
	at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.startWebServer(ServletWebServerApplicationContext.java:297) ~[spring-boot-2.2.4.RELEASE.jar:2.2.4.RELEASE]
	at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.finishRefresh(ServletWebServerApplicationContext.java:163) ~[spring-boot-2.2.4.RELEASE.jar:2.2.4.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:553) ~[spring-context-5.2.3.RELEASE.jar:5.2.3.RELEASE]
	at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:141) ~[spring-boot-2.2.4.RELEASE.jar:2.2.4.RELEASE]
	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:747) [spring-boot-2.2.4.RELEASE.jar:2.2.4.RELEASE]
	at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:397) [spring-boot-2.2.4.RELEASE.jar:2.2.4.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:315) [spring-boot-2.2.4.RELEASE.jar:2.2.4.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1226) [spring-boot-2.2.4.RELEASE.jar:2.2.4.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1215) [spring-boot-2.2.4.RELEASE.jar:2.2.4.RELEASE]
	at com.ts.feign.customer.CustomerApplication.main(CustomerApplication.java:14) [classes/:na]
Caused by: java.lang.IllegalArgumentException: standardService.connector.startFailed
	at org.apache.catalina.core.StandardService.addConnector(StandardService.java:231) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	at org.springframework.boot.web.embedded.tomcat.TomcatWebServer.addPreviouslyRemovedConnectors(TomcatWebServer.java:278) ~[spring-boot-2.2.4.RELEASE.jar:2.2.4.RELEASE]
	at org.springframework.boot.web.embedded.tomcat.TomcatWebServer.start(TomcatWebServer.java:197) ~[spring-boot-2.2.4.RELEASE.jar:2.2.4.RELEASE]
	... 10 common frames omitted
Caused by: org.apache.catalina.LifecycleException: Protocol handler start failed
	at org.apache.catalina.connector.Connector.startInternal(Connector.java:1008) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	at org.apache.catalina.core.StandardService.addConnector(StandardService.java:227) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	... 12 common frames omitted
Caused by: java.lang.IllegalArgumentException: Keystore was tampered with, or password was incorrect
	at org.apache.tomcat.util.net.AbstractJsseEndpoint.createSSLContext(AbstractJsseEndpoint.java:99) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	at org.apache.tomcat.util.net.AbstractJsseEndpoint.initialiseSsl(AbstractJsseEndpoint.java:71) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	at org.apache.tomcat.util.net.NioEndpoint.bind(NioEndpoint.java:217) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	at org.apache.tomcat.util.net.AbstractEndpoint.bindWithCleanup(AbstractEndpoint.java:1141) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	at org.apache.tomcat.util.net.AbstractEndpoint.start(AbstractEndpoint.java:1227) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	at org.apache.coyote.AbstractProtocol.start(AbstractProtocol.java:586) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	at org.apache.catalina.connector.Connector.startInternal(Connector.java:1005) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	... 14 common frames omitted
Caused by: java.io.IOException: Keystore was tampered with, or password was incorrect
	at sun.security.provider.JavaKeyStore.engineLoad(JavaKeyStore.java:780) ~[na:1.8.0_191]
	at sun.security.provider.JavaKeyStore$JKS.engineLoad(JavaKeyStore.java:56) ~[na:1.8.0_191]
	at sun.security.provider.KeyStoreDelegator.engineLoad(KeyStoreDelegator.java:224) ~[na:1.8.0_191]
	at sun.security.provider.JavaKeyStore$DualFormatJKS.engineLoad(JavaKeyStore.java:70) ~[na:1.8.0_191]
	at java.security.KeyStore.load(KeyStore.java:1445) ~[na:1.8.0_191]
	at org.apache.tomcat.util.security.KeyStoreUtil.load(KeyStoreUtil.java:69) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	at org.apache.tomcat.util.net.SSLUtilBase.getStore(SSLUtilBase.java:217) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	at org.apache.tomcat.util.net.SSLHostConfigCertificate.getCertificateKeystore(SSLHostConfigCertificate.java:206) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	at org.apache.tomcat.util.net.SSLUtilBase.getKeyManagers(SSLUtilBase.java:283) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	at org.apache.tomcat.util.net.SSLUtilBase.createSSLContext(SSLUtilBase.java:247) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	at org.apache.tomcat.util.net.AbstractJsseEndpoint.createSSLContext(AbstractJsseEndpoint.java:97) ~[tomcat-embed-core-9.0.30.jar:9.0.30]
	... 20 common frames omitted
Caused by: java.security.UnrecoverableKeyException: Password verification failed
	at sun.security.provider.JavaKeyStore.engineLoad(JavaKeyStore.java:778) ~[na:1.8.0_191]
	... 30 common frames omitted
```
这个报错很明显了，意思是密码错误，因为spring不会帮你自动解密密码（其实不用测试，就应该预料到的，因为它不知道你是通过什么算法加密的），它只会原封不动的使用该密码。

那么怎么来解决这个问题呢？
原理就是利用spring的扩展机制`EnvironmentPostProcessor`将环境中的加密变量解密，具体步骤如下：
##### 1、创建spring.factories
在当前项目的meta-inf目录下创建配置文件spring.factories，如果有了，就不用新创建了，
在该配置文件中增加如下配置：
```yml
org.springframework.boot.env.EnvironmentPostProcessor=com.ts.config.SafetyEncryptProcessor
```

##### 2、自定义解密类
其中`SafetyEncryptProcessor`是一个自定义类，实现如下：
```java
public class SafetyEncryptProcessor implements EnvironmentPostProcessor {
    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
        HashMap<String, Object> map = new HashMap<>();
        for (PropertySource<?> ps : environment.getPropertySources()) {
            if (ps instanceof OriginTrackedMapPropertySource) {
                boolean replace = false;
                OriginTrackedMapPropertySource source = (OriginTrackedMapPropertySource) ps;
                for (String name : source.getPropertyNames()) {
					// 判断是否存在加密参数，进行解密
				}
            }
        }
    }
}
```


### 客户端访问
因为是使用feign作为微服务之间的接口访问，因此这里就以feign为例进行讲解
关于feign的原理在之前的文章中已经讲解过了。
我们知道，通过feign调用服务由如下几种情况：
我们先来回顾下FeignClientFactoryBean类的getTarget方法的部分代码：
```java
if (!StringUtils.hasText(url)) {
	if (!name.startsWith("http")) {
		url = "http://" + name;
	}
	else {
		url = name;
	}
	url += cleanPath();
	return (T) loadBalance(builder, context,
			new HardCodedTarget<>(type, name, url));
}
if (StringUtils.hasText(url) && !url.startsWith("http")) {
	url = "http://" + url;
}
```

1、直接指定服务名
```java
@FeignClient(value = "ts-product",
    configuration = TsFeignClientsConfiguration.class)
public interface ControlFeign {
    @RequestMapping(value = "/status", method = RequestMethod.GET)
    String status();
}
```
从getTarget方法，可以知道，这种使用方式会走到`url = name;`这个分支，
也就是说，feign默认是以http方式进行通信的。

2、使用带schema的服务名
```java
@FeignClient(value = "https://ts-product",
    configuration = TsFeignClientsConfiguration.class)
public interface ControlFeign {
    @RequestMapping(value = "/status", method = RequestMethod.GET)
    String status();
}
```
从getTarget方法，可以知道，这种使用方式会走到`url = "http://" + name;`这个分支
也就是说，当指定了http或者https的时候，就会直接使用指定的schema

3、使用url
跟使用value类似，都分为默认的http和自定义的https。

通过上面几种情况的讲解，应该知道了，如果要让客户端采用https跟spring应用通信，就需要在value或者url中指定schema为https即可。

那么怎么配置client的证书信息呢？步骤如下：
##### 1、创建feignClient配置类
通过重定义feignClient即可，实例如下：
```java
@Configuration
public class TsFeignClientsConfiguration {
    @Bean
    public Feign.Builder feignBuilder(
        @Qualifier("cachingLBClientFactory") CachingSpringLoadBalancerFactory cachingFactory,
        SpringClientFactory clientFactory) throws Exception {
        return Feign.builder().client(feignClient(cachingFactory, clientFactory));
    }

    @Bean
    public Client feignClient(@Qualifier("cachingLBClientFactory") CachingSpringLoadBalancerFactory cachingFactory,
        SpringClientFactory clientFactory) throws Exception {
        // return new LoadBalancerFeignClient(new Client.Default(getSSLSocketFactory(), new NoopHostnameVerifier()),
        //     cachingFactory, clientFactory);
        return new LoadBalancerFeignClient(new Client.Default(null, new NoopHostnameVerifier()),
            cachingFactory, clientFactory);
    }

    private SSLSocketFactory getSSLSocketFactory() throws Exception {
        // 解密
        char[] allPassword = null;
        SSLContext sslContext = null;
        try {
            sslContext = SSLContextBuilder.create()
                .setKeyStoreType("JKS")
                .loadKeyMaterial(ResourceUtils.getFile("xxxxx"), allPassword, allPassword)
                .build();
        } catch (Exception e) {
            throw new Exception();
        }
        return sslContext.getSocketFactory();
    }
}
```

##### 2、指定自定义配置类
```java
@FeignClient(value = "https://ts-product",
    configuration = TsFeignClientsConfiguration.class)
public interface ControlFeign {
    @RequestMapping(value = "/status", method = RequestMethod.GET)
    String status();
}
```