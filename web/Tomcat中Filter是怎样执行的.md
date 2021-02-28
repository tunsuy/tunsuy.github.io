# Tomcat中Filter是怎样执行的

## 前言
Filter是什么？
Filter是servlet规范中定义的java web组件, 在所有支持java web的容器中都可以使用
它是位于前端请求到servlet之间的一系列过滤器，也可以称之为中间件，它主要是对请求到达servlet之前做一些额外的动作：
* 1、权限控制
* 2、监控
* 3、日志管理
* 4、等等

这里涉及到两个接口：Filter和FilterChain
Filter和FilterChain密不可分, Filter可以实现依次调用正是因为有了FilterChain。

### 1、Filter接口
```java
public interface Filter {
	 // 容器创建的时候调用, 即启动tomcat的时候调用
	 public void init(FilterConfig filterConfig) throws ServletException;
	 // 由FilterChain调用, 并且传入FilterChain本身, 最后回调FilterChain的doFilter()方法
	 public void doFilter(ServletRequest request, ServletResponse response,
			 FilterChain chain) throws IOException, ServletException;
	 // 容器销毁的时候调用, 即关闭tomcat的时候调用
	 public void destroy();
 }
```

### 2、FilterChain接口
```java
public interface FilterChain {
	 // 由Filter.doFilter()中的chain.doFilter调用
	 public void doFilter(ServletRequest request, ServletResponse response)
		 throws IOException, ServletException;
}
```

## 执行流程
在前面的文章中，我们知道，tomcat启动会执行`StandardWrapperValve.java`类的`invoke`方法：
```java
public final void invoke(Request request, Response response){
	......
	MessageBytes requestPathMB = request.getRequestPathMB();
	DispatcherType dispatcherType = DispatcherType.REQUEST;
	if (request.getDispatcherType()==DispatcherType.ASYNC) dispatcherType = DispatcherType.ASYNC;
	request.setAttribute(Globals.DISPATCHER_TYPE_ATTR,dispatcherType);
	request.setAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR,
			requestPathMB);
	// Create the filter chain for this request
	ApplicationFilterChain filterChain =
			ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);

	// Call the filter chain for this request
	// NOTE: This also calls the servlet's service() method
	Container container = this.container;
	try {
		if ((servlet != null) && (filterChain != null)) {
			// Swallow output if needed
			if (context.getSwallowOutput()) {
				try {
					SystemLogHandler.startCapture();
					if (request.isAsyncDispatching()) {
						request.getAsyncContextInternal().doInternalDispatch();
					} else {
						filterChain.doFilter(request.getRequest(),
								response.getResponse());
					}
				} finally {
					String log = SystemLogHandler.stopCapture();
					if (log != null && log.length() > 0) {
						context.getLogger().info(log);
					}
				}
			} else {
				if (request.isAsyncDispatching()) {
					request.getAsyncContextInternal().doInternalDispatch();
				} else {
					filterChain.doFilter
						(request.getRequest(), response.getResponse());
				}
			}

		}
	} catch (ClientAbortException | CloseNowException e) {
	
	}
	......
}
```
上面的代码做了如下一些动作：
* 1、每次请求过来都会创建一个过滤器链(filterChain)，并把待执行的servlet对象存放到过滤器链中。对于每个url，对应的filter个数都是不固定的，filterchain需要保存每个请求所对应的一个filter数组，以及调用到的filter的position，以便继续向下调用filter。
* 2、创建了filterChain之后，就开始执行doFilter进行请求的链式处理。

### 1、创建filterChain
下面我们具体来看看filterChain是怎么创建的
```java
public static ApplicationFilterChain createFilterChain(ServletRequest request,
            Wrapper wrapper, Servlet servlet) {

	// If there is no servlet to execute, return null
	if (servlet == null)
		return null;

	// Create and initialize a filter chain object
	ApplicationFilterChain filterChain = null;
	if (request instanceof Request) {
		Request req = (Request) request;
		if (Globals.IS_SECURITY_ENABLED) {
			// Security: Do not recycle
			filterChain = new ApplicationFilterChain();
		} else {
			filterChain = (ApplicationFilterChain) req.getFilterChain();
			if (filterChain == null) {
				filterChain = new ApplicationFilterChain();
				req.setFilterChain(filterChain);
			}
		}
	} else {
		// Request dispatcher in use
		filterChain = new ApplicationFilterChain();
	}

	filterChain.setServlet(servlet);
	filterChain.setServletSupportsAsync(wrapper.isAsyncSupported());

	// Acquire the filter mappings for this Context
	StandardContext context = (StandardContext) wrapper.getParent();
	FilterMap filterMaps[] = context.findFilterMaps();

	// If there are no filter mappings, we are done
	if ((filterMaps == null) || (filterMaps.length == 0))
		return filterChain;

	// Acquire the information we will need to match filter mappings
	DispatcherType dispatcher =
			(DispatcherType) request.getAttribute(Globals.DISPATCHER_TYPE_ATTR);

	String requestPath = null;
	Object attribute = request.getAttribute(Globals.DISPATCHER_REQUEST_PATH_ATTR);
	if (attribute != null){
		requestPath = attribute.toString();
	}

	String servletName = wrapper.getName();

	// Add the relevant path-mapped filters to this filter chain
	for (FilterMap filterMap : filterMaps) {
		if (!matchDispatcher(filterMap, dispatcher)) {
			continue;
		}
		if (!matchFiltersURL(filterMap, requestPath))
			continue;
		ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
				context.findFilterConfig(filterMap.getFilterName());
		if (filterConfig == null) {
			// FIXME - log configuration problem
			continue;
		}
		filterChain.addFilter(filterConfig);
	}

	// Add filters that match on servlet name second
	for (FilterMap filterMap : filterMaps) {
		if (!matchDispatcher(filterMap, dispatcher)) {
			continue;
		}
		if (!matchFiltersServlet(filterMap, servletName))
			continue;
		ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
				context.findFilterConfig(filterMap.getFilterName());
		if (filterConfig == null) {
			// FIXME - log configuration problem
			continue;
		}
		filterChain.addFilter(filterConfig);
	}

	// Return the completed filter chain
	return filterChain;
}
```
上面的代码做了一下几件事：
* 1、把要执行的servlet存放到过滤器链中。
* 2、如果没有配置过滤器则return一个空的过滤器链（只包含上面设置的servlet）。
* 3、如果配置url-pattern过滤器，则把匹配的过滤器加入到过滤器链中
* 4、如果配置servlet-name过滤器，则把匹配的过滤器加入到过滤器链中
##### 注意: filterChain.addFilter()顺序与web.xml中定义的Filter顺序一致，所以过滤器的执行顺序是按定义的上下顺序决定的。

### 2、执行dofilter
创建了chain之后，就开始执行链式请求了，具体的逻辑如下：
```java
private void internalDoFilter(ServletRequest request,
                                  ServletResponse response)
        throws IOException, ServletException {

	// Call the next filter if there is one
	if (pos < n) {
		ApplicationFilterConfig filterConfig = filters[pos++];
		try {
			Filter filter = filterConfig.getFilter();

			if (request.isAsyncSupported() && "false".equalsIgnoreCase(
					filterConfig.getFilterDef().getAsyncSupported())) {
				request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR, Boolean.FALSE);
			}
			if( Globals.IS_SECURITY_ENABLED ) {
				final ServletRequest req = request;
				final ServletResponse res = response;
				Principal principal =
					((HttpServletRequest) req).getUserPrincipal();

				Object[] args = new Object[]{req, res, this};
				SecurityUtil.doAsPrivilege ("doFilter", filter, classType, args, principal);
			} else {
				filter.doFilter(request, response, this);
			}
		} catch (IOException | ServletException | RuntimeException e) {
			throw e;
		} catch (Throwable e) {
			e = ExceptionUtils.unwrapInvocationTargetException(e);
			ExceptionUtils.handleThrowable(e);
			throw new ServletException(sm.getString("filterChain.filter"), e);
		}
		return;
	}

	// We fell off the end of the chain -- call the servlet instance
	try {
		if (ApplicationDispatcher.WRAP_SAME_OBJECT) {
			lastServicedRequest.set(request);
			lastServicedResponse.set(response);
		}

		if (request.isAsyncSupported() && !servletSupportsAsync) {
			request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR,
					Boolean.FALSE);
		}
		// Use potentially wrapped request from this point
		if ((request instanceof HttpServletRequest) &&
				(response instanceof HttpServletResponse) &&
				Globals.IS_SECURITY_ENABLED ) {
			final ServletRequest req = request;
			final ServletResponse res = response;
			Principal principal =
				((HttpServletRequest) req).getUserPrincipal();
			Object[] args = new Object[]{req, res};
			SecurityUtil.doAsPrivilege("service",
									   servlet,
									   classTypeUsedInService,
									   args,
									   principal);
		} else {
			servlet.service(request, response);
		}
	} catch (IOException | ServletException | RuntimeException e) {
		throw e;
	} catch (Throwable e) {
		e = ExceptionUtils.unwrapInvocationTargetException(e);
		ExceptionUtils.handleThrowable(e);
		throw new ServletException(sm.getString("filterChain.servlet"), e);
	} finally {
		if (ApplicationDispatcher.WRAP_SAME_OBJECT) {
			lastServicedRequest.set(null);
			lastServicedResponse.set(null);
		}
	}
}
```
上面的代码逻辑如下：
* 1、通过position索引判断是否执行完了所有的filter
* 2、如果没有，取出当前待执行的索引filter，调用其doFilter方法，在上面的接口说明中，我们看到，所有的filter类都继承了filter接口，都实现了dofilter方法；我们也注意到，该方法接收一个filterChain对象。在这段代码中，`filter.doFilter(request, response, this);`可以看到，将自身引用传递进去了，那么各个filter在dofilter的方法中，可以根据自身业务需要，来判断是否需要继续进行下面的filter链式执行，如果需要，就执行`filterChain.doFilter`方法，此时就又回到了此代码中。如果反复
* 3、如果执行完了所有的filter，则开始执行servlet业务模块`servlet.service(request, response);`