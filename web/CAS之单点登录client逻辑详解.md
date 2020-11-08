# CAS之单点登录client逻辑详解

关于单点登录SSO的原理，我在之前的文章中已经有详细的讲解，大家可以去看看历史文章。
今天这里主要是说下具体的实现逻辑，这里是基于java的，使用到了cas-client这个库。

Cas Client主要有四个核心过滤器：
* 1、AuthenticationFilter
* 2、TicketValidationFilter
* 3、HttpServletRequestWrapperFilter
* 4、AssertionThreadLocalFilter

下面会逐一的详细说明

## AuthenticationFilter
AuthenticationFilter用来拦截所有的请求，用以判断用户是否需要通过Cas Server进行认证，如果需要则将跳转到Cas Server的登录页面。如果不需要进行登录认证，则请求会继续往下执行。
AuthenticationFilter有两个用户必须指定的参数:
* 1、casServerLoginUrl: Cas Server登录地址的，
* 2、serverName或service: 指定认证成功后需要跳转地址的。service和serverName只需要指定一个就可以了。当两者都指定了，参数service将具有更高的优先级，即将以service指定的参数值为准。service和serverName的区别在于service指定的是一个确定的URL，认证成功后就会确切的跳转到service指定的URL；而serverName则是用来指定主机名，其格式为{protocol}:{hostName}:{port}，如：https://localhost:8443，当指定的是serverName时，AuthenticationFilter将会把它附加上当前请求的URI，以及对应的查询参数来构造一个确定的URL，如指定serverName为“http://localhost”，而当前请求的URI为“/app”，查询参数为“a=b&b=c”，则对应认证成功后的跳转地址将为“http://localhost/app?a=b&b=c”。

除了上述必须指定的参数外，AuthenticationFilter还可以指定如下可选参数：
* 1、renew：当指定renew为true时，在请Cas Server时将带上参数“renew=true”，默认为false。
* 2、gateway：指定gateway为true时，在请求Cas Server时将带上参数“gateway=true”，默认为false。
* 3、artifactParameterName：指定ticket对应的请求参数名称，默认为ticket。
* 4、serviceParameterName：指定service对应的请求参数名称，默认为service。

### 1、配置文件
一般来说，我们会像下面这样配置相关信息，web.xml文件配置如下：
```sh
<context-param>
	<param-name>serverName</param-name>
	<param-value>${sso.address.client}</param-value>
</context-param>
<context-param>
	<param-name>casServerUrlPrefix</param-name>
	<param-value>${sso.address.server}/unisso</param-value>
</context-param>
<context-param>
	<param-name>casServerLoginUrl</param-name>
	<param-value>${sso.address.server}/unisso/login</param-value>
</context-param>
	
<filter>
	<filter-name>ssoAuthFilter</filter-name>
	<filter-class>com.huawei.ism.drm.web.filter.SSOIntegrateFilter</filter-class>
	<init-param>
		<param-name>wrappedFilterClass</param-name>
		<param-value>org.jasig.cas.client.authentication.AuthenticationFilter</param-value>
	</init-param>
</filter>
```
上面的配置就对应了上面说的那些参数，大家可以感受下。

### 2、代码分析
下面会说下具体的代码实现，代码定义如下：
```java
public class AuthenticationFilter extends AbstractCasFilter {
    /**
     * The URL to the CAS Server login.
     */
    private String casServerLoginUrl;

    /**
     * Whether to send the renew request or not.
     */
    private boolean renew = false;

    /**
     * Whether to send the gateway request or not.
     */
    private boolean gateway = false;

    /**
     * The method used by the CAS server to send the user back to the application.
     */
    private String method;

    private GatewayResolver gatewayStorage = new DefaultGatewayResolverImpl();

    private AuthenticationRedirectStrategy authenticationRedirectStrategy = new DefaultAuthenticationRedirectStrategy();

    private UrlPatternMatcherStrategy ignoreUrlPatternMatcherStrategyClass = null;

    private static final Map<String, Class<? extends UrlPatternMatcherStrategy>> PATTERN_MATCHER_TYPES =
        new HashMap<String, Class<? extends UrlPatternMatcherStrategy>>();

    static {
        PATTERN_MATCHER_TYPES.put("CONTAINS", ContainsPatternUrlPatternMatcherStrategy.class);
        PATTERN_MATCHER_TYPES.put("REGEX", RegexUrlPatternMatcherStrategy.class);
        PATTERN_MATCHER_TYPES.put("FULL_REGEX", EntireRegionRegexUrlPatternMatcherStrategy.class);
        PATTERN_MATCHER_TYPES.put("EXACT", ExactUrlPatternMatcherStrategy.class);
    }

    public AuthenticationFilter() {
        this(Protocol.CAS2);
    }

    protected AuthenticationFilter(final Protocol protocol) {
        super(protocol);
    }
}
```
上面是对应的参数定义。

对于一个filter来说，我们一般就主要看下dofilter方法，代码如下：
```java
public final void doFilter(final ServletRequest servletRequest,
		final ServletResponse servletResponse, final FilterChain filterChain)
		throws IOException, ServletException {
	final HttpServletRequest request = (HttpServletRequest) servletRequest;
	final HttpServletResponse response = (HttpServletResponse) servletResponse;
	final HttpSession session = request.getSession(false);
	//从session中获取名为"_const_cas_assertion_"的Assertion
	final Assertion assertion = (session != null) ? (Assertion) session.getAttribute(CONST_CAS_ASSERTION) : null;
	//如果存在，则说明已经登录，本过滤器处理完成，处理下个过滤器
	if (assertion != null) {
		filterChain.doFilter(request, response);
		return;
	}
	//生成serviceUrl
	final String serviceUrl = constructServiceUrl(request, response);
	//从request中获取参数ticket
	final String ticket = CommonUtils.safeGetParameter(request, getArtifactParameterName());
	final boolean wasGatewayed = this.gatewayStorage.hasGatewayedAlready(request, serviceUrl);
	//如果ticket不为空，本过滤器处理完成，处理下个过滤器
	if ((CommonUtils.isNotBlank(ticket)) || (wasGatewayed)) {
		filterChain.doFilter(request, response);
		return;
	}
 
	this.log.debug("no ticket and no assertion found");
 
	final String modifiedServiceUrl;
	if (this.gateway) {
		this.log.debug("setting gateway attribute in session");
		modifiedServiceUrl = this.gatewayStorage.storeGatewayInformation(request, serviceUrl);
	} else {
		modifiedServiceUrl = serviceUrl;
	}
 
	if (this.log.isDebugEnabled()) {
		this.log.debug("Constructed service url: " + modifiedServiceUrl);
	}
 
	//生成重定向URL
	final String urlToRedirectTo = CommonUtils.constructRedirectUrl(this.casServerLoginUrl, getServiceParameterName(),
			modifiedServiceUrl, this.renew, this.gateway);
 
	if (this.log.isDebugEnabled()) {
		this.log.debug("redirecting to \"" + urlToRedirectTo + "\"");
	}
	//跳转到CAS服务器的登录页面
	response.sendRedirect(urlToRedirectTo);
}
```
上面的逻辑总体如下：
* 1、从session中获取名为“_const_cas_assertion_”的assertion对象，判断assertion是否存在，如果存在，说明已经登录，执行下一个过滤器。如果不存在，执行第2步。
* 2、从request中获取票据参数ticket，判断ticket是否为空，如果不为空执行下一个过滤器。如果为空，执行第3步。
* 3、生成重定向URL，跳转到单点登录服务器，显示登录页面，此时第一个过滤器执行完成。


## TicketValidationFilter
在请求通过AuthenticationFilter的认证之后，如果请求中携带了参数ticket则将会由TicketValidationFilter来对携带的ticket进行校验。TicketValidationFilter只是对验证ticket的这一类Filter的统称，其并不对应Cas Client中的一个具体类型。Cas Client中有多种验证ticket的Filter，都继承自AbstractTicketValidationFilter，它们的验证逻辑都是一致的，都有AbstractTicketValidationFilter实现，所不同的是使用的TicketValidator不一样。默认是Cas10TicketValidationFilter。

可选参数包括：
* 1、redirectAfterValidation：表示是否验证通过后重新跳转到该URL，但是不带参数ticket，默认为true。
* 2、useSession：在验证ticket成功后会生成一个Assertion对象，如果useSession为true，则会将该对象存放到Session中。如果为false，则要求每次请求都需要携带ticket进行验证，显然useSession为false跟redirectAfterValidation为true是冲突的。默认为true。
* 3、exceptionOnValidationFailure：表示ticket验证失败后是否需要抛出异常，默认为true。
* 4、renew：当值为true时将发送“renew=true”到Cas Server，默认为false。
  代码定义如下：
```java
public abstract class AbstractTicketValidationFilter extends AbstractCasFilter {

    /** The TicketValidator we will use to validate tickets. */
    private TicketValidator ticketValidator;

    /**
     * Specify whether the filter should redirect the user agent after a
     * successful validation to remove the ticket parameter from the query
     * string.
     */
    private boolean redirectAfterValidation = true;

    /** Determines whether an exception is thrown when there is a ticket validation failure. */
    private boolean exceptionOnValidationFailure = false;

    /**
     * Specify whether the Assertion should be stored in a session
     * attribute {@link AbstractCasFilter#CONST_CAS_ASSERTION}.
     */
    private boolean useSession = true;
	...
}
```
cas-client库提供了现成的类可以直接使用，如下：
```java
public class Cas20ProxyReceivingTicketValidationFilter extends AbstractTicketValidationFilter {}
```
一般也可以继承该类，扩展自己的逻辑

下面我们主要来看下dofilter方法：
```java
public final void doFilter(final ServletRequest servletRequest,
		final ServletResponse servletResponse, final FilterChain filterChain)
		throws IOException, ServletException {
	if (!preFilter(servletRequest, servletResponse, filterChain)) {
		return;
	}
 
	final HttpServletRequest request = (HttpServletRequest) servletRequest;
	final HttpServletResponse response = (HttpServletResponse) servletResponse;
	//从request中获取ticket
	final String ticket = CommonUtils.safeGetParameter(request, getArtifactParameterName());
	//ticket不为空，验证ticket，否则本过滤器处理完成，处理下个过滤器
	if (CommonUtils.isNotBlank(ticket)) {
		if (this.log.isDebugEnabled()) {
			this.log.debug("Attempting to validate ticket: " + ticket);
		}
		try {
			//验证ticket并产生Assertion对象，错误抛出TicketValidationException异常
			final Assertion assertion = this.ticketValidator.validate(ticket, constructServiceUrl(request, response));
 
			if (this.log.isDebugEnabled()) {
				this.log.debug("Successfully authenticated user: " + assertion.getPrincipal().getName());
			}
			//request设置assertion
			request.setAttribute(CONST_CAS_ASSERTION, assertion);
			//session设置assertion
			if (this.useSession) {
				request.getSession().setAttribute(CONST_CAS_ASSERTION, assertion);
			}
			
			// 成功回调
			onSuccessfulValidation(request, response, assertion);
 
			if (this.redirectAfterValidation) {
				this.log.debug("Redirecting after successful ticket validation.");
				response.sendRedirect(constructServiceUrl(request, response));
				return;
			}
		} catch (final TicketValidationException e) {
			logger.debug(e.getMessage(), e);

			// 失败回调
			onFailedValidation(request, response);

			if (this.exceptionOnValidationFailure) {
				throw new ServletException(e);
			}

			response.sendError(HttpServletResponse.SC_FORBIDDEN, e.getMessage());

			return;
		}
	}
	//本过滤器处理完成，处理下个过滤器
	filterChain.doFilter(request, response);
}
```
假设当执行完第一个过滤器后，跳转到CAS服务器端的登录页面，输入用户名和密码，验证通过后。CAS服务器端会生成ticket，并将ticket作为重新跳转到应用系统的参数。此时又进入第一个过滤器AuthenticationFilter，由于存在ticket参数，进入到第二个过滤器TicketValidationFilter，执行以下操作：
* 1、从request获取ticket参数，如果ticket为空，继续处理下一个过滤器。如果参数不为空，验证ticket参数的合法性。
* 2、验证ticket：`this.ticketValidator.validate`方法通过httpClient访问CAS服务器端验证ticket是否正确，并返回assertion对象。如果验证失败，抛出异常，跳转到错误页面。如果验证成功，session会以"_const_cas_assertion_"的名称保存assertion对象，继续处理下一个过滤器。
  这里的`ticketValidator`是个接口，我们可以基于自己的业务实现它
```java
public interface TicketValidator {

    /**
     * Attempts to validate a ticket for the provided service.
     *
     * @param ticket the ticket to attempt to validate.
     * @param service the service this ticket is valid for.
     * @return an assertion from the ticket.
     * @throws TicketValidationException if the ticket cannot be validated.
     *
     */
    Assertion validate(String ticket, String service) throws TicketValidationException;
}
```
* 3、成功之后的回调：`onSuccessfulValidation`是个开放接口，提供给实现方自己实现，所以一般如果我们继承的`AbstractTicketValidationFilter`类，一般都需要实现该接口。同理，也有失败的回调。

## HttpServletRequestWrapperFilter
HttpServletRequestWrapperFilter用于将每一个请求对应的HttpServletRequest封装为其内部定义的CasHttpServletRequestWrapper。

该封装类将利用之前保存在Session或request中的Assertion对象重写HttpServletRequest的getUserPrincipal()、getRemoteUser()和isUserInRole()方法。

这样在我们的应用中就可以非常方便的从HttpServletRequest中获取到用户的相关信息。

注：这个过滤器是可选的。

## AssertionThreadLocalFilter
AssertionThreadLocalFilter是为了方便用户在应用的其它地方获取Assertion对象，其会将当前的Assertion对象存放到当前的线程变量中，那么以后用户在程序的任何地方都可以从线程变量中获取当前Assertion，无需再从Session或request中进行解析。

该线程变量是由AssertionHolder持有的，我们在获取当前的Assertion时也只需要通过AssertionHolder的getAssertion()方法获取即可，如：

```java
   Assertion assertion = AssertionHolder.getAssertion();
```
从命名我们就知道，里面是利用了ThreadLocal机制来实现的。

注：这个过滤器是可选的。