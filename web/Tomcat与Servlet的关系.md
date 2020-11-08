# Tomcat与Servlet的关系

## 什么是servlet
什么是servlet，大部分人可能都会说是规范，没错，我觉得从大的方面来说，它是一个规范，只要一个web容器遵循了该规范，我们实现的servlet就可以继承到太容器中。
但是具体来说，也可以说它只是一个接口，它定义了一系列的接口，如下所示：
```java
public interface Servlet {
    void init(ServletConfig var1) throws ServletException;

    ServletConfig getServletConfig();

    void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;

    String getServletInfo();

    void destroy();
}
```

servlet接口定义的是一套处理网络请求的规范，所有实现servlet的类，都需要实现它那五个方法，其中最主要的是两个生命周期方法`init()和destroy()`，还有一个处理请求的`service()`，
也就是说，所有实现servlet接口的类，或者说，所有想要处理网络请求的类，都需要回答这三个问题：
* 你初始化时要做什么
* 你销毁时要做什么
* 你接受到请求时要做什么
  这是Java给的一种规范！

写过servlet的人应该都有印象，我们一般都是继承httpservlet，为什么呢？
```java
public abstract class GenericServlet implements Servlet, ServletConfig, Serializable {
    ......
}

public abstract class HttpServlet extends GenericServlet {
    ......
}
```
从继承关系我们就知道了，因为我们知道，对一个request的处理，很多都是公共的逻辑：比如解析请求，判断是哪种请求类型（GET、POST等等），然后根据不同的类型执行不同的业务逻辑，
因此，为了统一这些公共逻辑的处理，serlet就为我们提供了这样一个类让我们继承，我们只需要实现do_get、do_post等业务方法即可，其他的交给父类处理。

那么servlet是怎么被调用的呢？下面我们就来看看

## tomcat对servlet的调用
servlet具体来说只是一个接口类，定义了相关的接口，需要实现该接口处理自己的业务逻辑。既然是接口类，那么必然要存在于容器中，通过容器框架来进行调用，这里最常用的容器大家肯定知道，那就是tomcat。
tomcat才是与客户端直接打交道的，它监听了端口，请求过来后，根据url等信息，确定要将请求交给哪个servlet去处理，然后调用那个servlet的service方法，service方法返回一个response对象，tomcat再把这个response返回给客户端。
注：这里需要说明一点的是，servlet并不是只能适用于tomcat中，只要是遵循了servlet规范（这里指的是可以处理servlet接口类）的web服务器都可以。

下面我们来看看tomcat是如何调用servlet的？

### 加载web.xml
在Tomcat进行加载web.xml配置的时候，就是在org.apache.catalina.deploy.WebXml类的 configureContext 方法中，可以看到很多的setXX,和addXX的方法，把文件解析后表示Servlet、Listener、Filter 的配置信息都与表示web应用的Context对象关联起来
```java
public void addServlet(ServletDef servletDef) {
        this.servlets.put(servletDef.getServletName(), servletDef);
        if (this.overridable) {
            servletDef.setOverridable(this.overridable);
        }

    }
```

`需要注意的是，上面的add方法仅仅是把配置信息保存到Context的相应实例变量中，而真正的在请求中响应的 Servlet、Listener、Filter 的实例并没有构造出来。`
在StandardContext也就是web应用被Host构建的时候，会发布事件，最终会调用解析加载web.xml的方法。然后，这里会把解析的配置封装类关联到StandardContext中去。

`org.apache.catalina.core.StandardContext`类的 startInternal 方法中
```java
if (ok && !this.listenerStart()) {
	log.error(sm.getString("standardContext.listenerFail"));
	ok = false;
}

if (ok) {
	this.checkConstraintsForUncoveredMethods(this.findConstraints());
}

try {
	Manager manager = this.getManager();
	if (manager instanceof Lifecycle) {
		((Lifecycle)manager).start();
	}
} catch (Exception var18) {
	log.error(sm.getString("standardContext.managerFail"), var18);
	ok = false;
}

if (ok && !this.filterStart()) {
	log.error(sm.getString("standardContext.filterFail"));
	ok = false;
}

if (ok && !this.loadOnStartup(this.findChildren())) {
	log.error(sm.getString("standardContext.servletFail"));
	ok = false;
}

super.threadStart();
```
调用了listenerStart ，filterStart ，loadOnStartup 方法，这里即触发 Listener、Filter、Servlet 真正对象的构造。

### servlet实例构造
这里具体分析下servlet的构造
```java
public boolean loadOnStartup(Container children[]) {

    // Collect "load on startup" servlets that need to be initialized
    TreeMap<Integer, ArrayList<Wrapper>> map =
        new TreeMap<Integer, ArrayList<Wrapper>>();
    for (int i = 0; i < children.length; i++) {
        Wrapper wrapper = (Wrapper) children[i];
        int loadOnStartup = wrapper.getLoadOnStartup();
        if (loadOnStartup < 0)
            continue;
        Integer key = Integer.valueOf(loadOnStartup);
        ArrayList<Wrapper> list = map.get(key);
        if (list == null) {
            list = new ArrayList<Wrapper>();
            map.put(key, list);
        }
        list.add(wrapper);
    }

    // Load the collected "load on startup" servlets
    for (ArrayList<Wrapper> list : map.values()) {
        for (Wrapper wrapper : list) {
            try {
                wrapper.load();
            } catch (ServletException e) {
                getLogger().error(sm.getString("standardContext.loadOnStartup.loadException",
                      getName(), wrapper.getName()), StandardWrapper.getRootCause(e));
                if(getComputedFailCtxIfServletStartFails()) {
                    return false;
                }
            }
        }
    }
    return true;
}
```
这里会对Servlet进行初始化实例操作代码是：wrapper.load();
`需要注意的是：这里的加载只是针对配置了 load-on-startup 属性的 Servlet 而言，其它一般 Servlet 的加载和初始化会推迟到真正请求访问 web 应用而第一次调用该 Servlet 时`

### servlet的调用
根据Tomcat一次请求的流程，可以知道请求会在容器Engine、Host、Context、Wrapper 各级组件中匹配，并且在他们的管道中流转。最终是会适配到一个StandardWrapper 的基础阀的-org.apache.catalina.core.StandardWrapperValve的 invoke 方法。
```java
final class StandardWrapperValve extends ValveBase {
@Override
public final void invoke(Request request, Response response)
    throws IOException, ServletException {

	......

    // Allocate a servlet instance to process this request
    try {
        if (!unavailable) {
            servlet = wrapper.allocate();  
        }
    } catch (UnavailableException e) {
        ......
    }

	......
	
    // Create the filter chain for this request 
    ApplicationFilterFactory factory =
        ApplicationFilterFactory.getInstance();
     ApplicationFilterChain filterChain =
        factory.createFilterChain(request, wrapper, servlet);

    // Reset comet flag value after creating the filter chain
    request.setComet(false);

    // Call the filter chain for this request
    // NOTE: This also calls the servlet's service() method
    try {
        if ((servlet != null) && (filterChain != null)) {
            // Swallow output if needed
            if (context.getSwallowOutput()) {
                try {
                    SystemLogHandler.startCapture();
                    if (request.isAsyncDispatching()) {
                        request.getAsyncContextInternal().doInternalDispatch();
                    } else if (comet) {
                        filterChain.doFilterEvent(request.getEvent());
                        request.setComet(true);
                    } else {
                        filterChain.doFilter(request.getRequest(),
                                response.getResponse());
                    }
                } finally {.......}
            } else {
                if (request.isAsyncDispatching()) {
                    request.getAsyncContextInternal().doInternalDispatch();
                } else if (comet) {
                    request.setComet(true);
                    filterChain.doFilterEvent(request.getEvent());
                } else {
                    filterChain.doFilter    
                        (request.getRequest(), response.getResponse());
                }
            }
        }
    } catch (ClientAbortException e) {......}

    ......
}

```
这里会构建filterChain-过滤器链，用于执行一次请求所经过的相应Filter。接着会链式调用filterChain 的doFilter方法。 来看一下doFilter的具体实现：
```java
//final class ApplicationFilterChain implements FilterChain, CometFilterChain {
public void doFilter(ServletRequest request, ServletResponse response)
    throws IOException, ServletException {

    if( Globals.IS_SECURITY_ENABLED ) {
        final ServletRequest req = request;
        final ServletResponse res = response;
        try {
            java.security.AccessController.doPrivileged(
                new java.security.PrivilegedExceptionAction<Void>() {
                    @Override
                    public Void run() 
                        throws ServletException, IOException {
                        internalDoFilter(req,res);  //********
                        return null;
                    }
                }
            );
        } catch( PrivilegedActionException pe) {......}
    } else {
        internalDoFilter(request,response);//********
    }
}

private void internalDoFilter(ServletRequest request, 
                                  ServletResponse response)
        throws IOException, ServletException {

        // Call the next filter if there is one
        if (pos < n) {
            ApplicationFilterConfig filterConfig = filters[pos++];
            Filter filter = null;
            try {
                filter = filterConfig.getFilter();
                support.fireInstanceEvent(InstanceEvent.BEFORE_FILTER_EVENT,
                                          filter, request, response);
                if (request.isAsyncSupported() && "false".equalsIgnoreCase(
                        filterConfig.getFilterDef().getAsyncSupported())) {
                    request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR,
                            Boolean.FALSE);
                }
                if( Globals.IS_SECURITY_ENABLED ) {
                    final ServletRequest req = request;
                    final ServletResponse res = response;
                    Principal principal = 
                        ((HttpServletRequest) req).getUserPrincipal();

                    Object[] args = new Object[]{req, res, this};
                    SecurityUtil.doAsPrivilege
                        ("doFilter", filter, classType, args, principal);
                } else {  
                    filter.doFilter(request, response, this);  //***就是这个***
                }

                support.fireInstanceEvent(InstanceEvent.AFTER_FILTER_EVENT,
                                          filter, request, response);
            } catch (IOException e) {......}
            return;
        }
		
        // We fell off the end of the chain -- call the servlet instance
        try {
            if (ApplicationDispatcher.WRAP_SAME_OBJECT) {
                lastServicedRequest.set(request);
                lastServicedResponse.set(response);
            }

            support.fireInstanceEvent(InstanceEvent.BEFORE_SERVICE_EVENT,
                                      servlet, request, response);
            if (request.isAsyncSupported()
                    && !support.getWrapper().isAsyncSupported()) {
                request.setAttribute(Globals.ASYNC_SUPPORTED_ATTR,
                        Boolean.FALSE);
            }
            // Use potentially wrapped request from this point
            if ((request instanceof HttpServletRequest) &&
                (response instanceof HttpServletResponse)) {
                    
                if( Globals.IS_SECURITY_ENABLED ) {
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
            } else {
                servlet.service(request, response);
            }
            support.fireInstanceEvent(InstanceEvent.AFTER_SERVICE_EVENT,
                                      servlet, request, response);
        } catch (IOException e) {......} finally {
            if (ApplicationDispatcher.WRAP_SAME_OBJECT) {
                lastServicedRequest.set(null);
                lastServicedResponse.set(null);
            }
        }
    }
```
前面的dofilter先不看，这里我们注意到：`servlet.service(request, response);`这样一句
看到这里是不是就清楚了，这里的service方法就是前面我们说的servlet接口的入口方法，也就是走到了我们的自定义的servlet类的业务方法里面去了。