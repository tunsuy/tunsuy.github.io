# CAS之单点登出逻辑详解

单点登出功能跟单点登录功能是相对应的，旨在通过Cas Server的登出使所有的Cas Client都登出。
Cas Server的登出是通过请求“/logout”发生的，即如果你的Cas Server部署的访问路径为“https://localhost:8443/cas”时，通过访问“https://localhost:8443/cas/logout”可以触发Cas Server的登出操作，进而触发Cas Client的登出。
在请求Cas Server的logout时，Cas Server会将客户端携带的TGC删除，同时回调该TGT对应的所有service，即所有的Cas Client。
Cas Client如果需要响应该回调，进而在Cas Client端进行登出操作的话就需要有对应的支持。

## 配置文件
具体来说，需要在Cas Client应用的web.xml文件中添加如下Filter和Listener。
```sh
<filter>
   <filter-name>CAS Single Sign Out Filter</filter-name>
   <filter-class>org.jasig.cas.client.session.SingleSignOutFilter</filter-class>
</filter>
<filter-mapping>
   <filter-name>CAS Single Sign Out Filter</filter-name>
   <url-pattern>/*</url-pattern>
</filter-mapping>
<listener>
    <listener-class>org.jasig.cas.client.session.SingleSignOutHttpSessionListener</listener-class>
</listener>
```
当然，也可以自定义filter和监听，通过该继承或者组合`SingleSignOutFilter`和`SingleSignOutHttpSessionListener`的方式

下面我们来具体看看这几个过滤器

## SingleSignOutFilter
下面我们来看下该过滤器的代码:
```java
@Override
public void doFilter(final ServletRequest servletRequest, final ServletResponse servletResponse,
					 final FilterChain filterChain) throws IOException, ServletException {
	final HttpServletRequest request = (HttpServletRequest) servletRequest;
	final HttpServletResponse response = (HttpServletResponse) servletResponse;

	/**
	 * <p>Workaround for now for the fact that Spring Security will fail since it doesn't call {@link #init(javax.servlet.FilterConfig)}.</p>
	 * <p>Ultimately we need to allow deployers to actually inject their fully-initialized {@link org.jasig.cas.client.session.SingleSignOutHandler}.</p>
	 */
	if (!this.handlerInitialized.getAndSet(true)) {
		HANDLER.init();
	}

	if (HANDLER.process(request, response)) {
		filterChain.doFilter(servletRequest, servletResponse);
	}
}
```
首先会调用handler的初始化方法，那么该handler是什么呢？就是对应的`SingleSignOutHander`类，init方法如下：
```java
public synchronized void init() {
	if (this.safeParameters == null) {
		CommonUtils.assertNotNull(this.artifactParameterName, "artifactParameterName cannot be null.");
		CommonUtils.assertNotNull(this.logoutParameterName, "logoutParameterName cannot be null.");
		CommonUtils.assertNotNull(this.sessionMappingStorage, "sessionMappingStorage cannot be null.");
		CommonUtils.assertNotNull(this.relayStateParameterName, "relayStateParameterName cannot be null.");

		if (this.artifactParameterOverPost) {
			this.safeParameters = Arrays.asList(this.logoutParameterName, this.artifactParameterName);
		} else {
			this.safeParameters = Collections.singletonList(this.logoutParameterName);
		}
	}
}
```
判断是否具有logoutParameterName参数指定的参数，默认参数名为logoutRequest

随后调用了process方法：
```java
public boolean process(final HttpServletRequest request, final HttpServletResponse response) {
	if (isTokenRequest(request)) {
		logger.trace("Received a token request");
		recordSession(request);
		return true;
	} 
	
	if (isLogoutRequest(request)) {
		logger.trace("Received a logout request");
		destroySession(request);
		return false;
	} 
	logger.trace("Ignoring URI for logout: {}", request.getRequestURI());
	return true;
}
```
可以看到有两处逻辑：
* 1、判断是否是正常的ticket请求
* 2、判断是否是logout登出请求

下面依次来解析。

### 1、ticket请求
判断是否具有artifactParameterName参数指定的参数，默认参数名为ticket。
如果有，就表示是一个正常的业务请求，进入该分支。可以看到有个recordSession方法，从名字就可以看出是保存session的，我们进入其中，看看代码：
```java
private void recordSession(final HttpServletRequest request) {
	final HttpSession session = request.getSession(this.eagerlyCreateSessions);

	if (session == null) {
		logger.debug("No session currently exists (and none created).  Cannot record session information for single sign out.");
		return;
	}

	final String token = CommonUtils.safeGetParameter(request, this.artifactParameterName, this.safeParameters);
	logger.debug("Recording session for token {}", token);

	try {
		this.sessionMappingStorage.removeBySessionById(session.getId());
	} catch (final Exception e) {
		// ignore if the session is already marked as invalid. Nothing we can do!
	}
	sessionMappingStorage.addSessionById(token, session);
}

```
大致逻辑就是：从request中取出session和ticket，先从存储中移除该sessionId对应的session，然后以新的tocket作为key将该session存储下来。
这里的`sessionMappingStorage`是什么，我们后面再讲。

### 2、logout登出请求
判断是否logout登出请求。
如果是，就表示是一个退出业务系统的请求，进入该分支。可以看到有个destroySession方法，从名字就可以看出是销毁session的，我们进入其中，看看代码：
```java
private void destroySession(final HttpServletRequest request) {
	String logoutMessage = CommonUtils.safeGetParameter(request, this.logoutParameterName, this.safeParameters);
	if (CommonUtils.isBlank(logoutMessage)) {
		logger.error("Could not locate logout message of the request from {}", this.logoutParameterName);
		return;
	}
	
	if (!logoutMessage.contains("SessionIndex")) {
		logoutMessage = uncompressLogoutMessage(logoutMessage);
	}
	
	logger.trace("Logout request:\n{}", logoutMessage);
	final String token = XmlUtils.getTextForElement(logoutMessage, "SessionIndex");
	if (CommonUtils.isNotBlank(token)) {
		final HttpSession session = this.sessionMappingStorage.removeSessionByMappingId(token);

		if (session != null) {
			final String sessionID = session.getId();
			logger.debug("Invalidating session [{}] for token [{}]", sessionID, token);

			try {
				session.invalidate();
			} catch (final IllegalStateException e) {
				logger.debug("Error invalidating session.", e);
			}
			this.logoutStrategy.logout(request);
		}
	}
}
```
首先从request中取出token，也就是ticket，然后根据ticket从存储中移除对应的session，同时将该session销毁，最后执行登出策略的回调接口。

那么这个登出请求是在什么时候会触发呢，也就是说是谁通知的业务系统呢？
这个是在你退出登录的任意ST客户端，经过cas server的时候，cas server会取得cookie里面的TGT数据，找到TGT中关联的所有ST对应的地址，也就是各个业务系统，向每个地址方式一个http请求，并传递logoutRequest参数。

## SessionMappingStorage
上面我们提到了，session的存储都是通过`sessionMappingStorage`来的，那么现在就来来看看这个是什么？
我们进入代码：
```java
public interface SessionMappingStorage {

    /**
     * Remove the HttpSession based on the mappingId.
     * 
     * @param mappingId the id the session is keyed under.
     * @return the HttpSession if it exists.
     */
    HttpSession removeSessionByMappingId(String mappingId);

    /**
     * Remove a session by its Id.
     * @param sessionId the id of the session.
     */
    void removeBySessionById(String sessionId);

    /**
     * Add a session by its mapping Id.
     * @param mappingId the id to map the session to.
     * @param session the HttpSession.
     */
    void addSessionById(String mappingId, HttpSession session);

}
```
可以看到，它是一个提供了三个方法的接口类，默认提供了一个hash方式的实现类：
```java
public final class HashMapBackedSessionMappingStorage implements SessionMappingStorage {

    /**
     * Maps the ID from the CAS server to the Session.
     */
    private final Map<String, HttpSession> MANAGED_SESSIONS = new HashMap<String, HttpSession>();

    /**
     * Maps the Session ID to the key from the CAS Server.
     */
    private final Map<String, String> ID_TO_SESSION_KEY_MAPPING = new HashMap<String, String>();

    private final Logger logger = LoggerFactory.getLogger(getClass());

    @Override
    public synchronized void addSessionById(final String mappingId, final HttpSession session) {
        ID_TO_SESSION_KEY_MAPPING.put(session.getId(), mappingId);
        MANAGED_SESSIONS.put(mappingId, session);

    }

    @Override
    public synchronized void removeBySessionById(final String sessionId) {
        logger.debug("Attempting to remove Session=[{}]", sessionId);

        final String key = ID_TO_SESSION_KEY_MAPPING.get(sessionId);

        if (logger.isDebugEnabled()) {
            if (key != null) {
                logger.debug("Found mapping for session.  Session Removed.");
            } else {
                logger.debug("No mapping for session found.  Ignoring.");
            }
        }
        MANAGED_SESSIONS.remove(key);
        ID_TO_SESSION_KEY_MAPPING.remove(sessionId);
    }

    @Override
    public synchronized HttpSession removeSessionByMappingId(final String mappingId) {
        final HttpSession session = MANAGED_SESSIONS.get(mappingId);

        if (session != null) {
            removeBySessionById(session.getId());
        }

        return session;
    }
}
```
也就是通过hashmap来实现内存中session的存储，大家可能就已经想到了，如果是分布式环境下，这个实现类就不合适了，需要自己实现，具体方案后面新开一篇文章详解。

## SingleSignOutHttpSessionListener
有时候，如果我们需要在系统退出之后，做一些额外的处理，则可以直接加上该监听，当然，也可以自定义一个HttpSessionListener接口类，在该类中关联该SingleSignOutHttpSessionListener类，同时增加一些额外的处理即可。
下面我们来看下该监听默认的实现：
```java
public final class SingleSignOutHttpSessionListener implements HttpSessionListener {

    private SessionMappingStorage sessionMappingStorage;

    @Override
    public void sessionCreated(final HttpSessionEvent event) {
        // nothing to do at the moment
    }

    @Override
    public void sessionDestroyed(final HttpSessionEvent event) {
        if (sessionMappingStorage == null) {
            sessionMappingStorage = getSessionMappingStorage();
        }
        final HttpSession session = event.getSession();
        sessionMappingStorage.removeBySessionById(session.getId());
    }

    /**
     * Obtains a {@link SessionMappingStorage} object. Assumes this method will always return the same
     * instance of the object.  It assumes this because it generally lazily calls the method.
     * 
     * @return the SessionMappingStorage
     */
    protected static SessionMappingStorage getSessionMappingStorage() {
        return SingleSignOutFilter.getSingleSignOutHandler().getSessionMappingStorage();
    }
}
```
很简单，也就是使用sessionMappingStorage进行session的销毁。