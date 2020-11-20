---
title: "Spring源码学习-SpringMVC中DispatcherServlet实现原理"
date: 2020-11-05T18:10:12+08:00
tags: ["Spring", "源码学习","SpringMVC","DispatcherServlet"]
draft: false
---

> Spring 版本：5.2.6.RELEASE，笔者的Spring源码学习笔记参考 郝佳 编著的《Spring源码深度解析》，推荐感兴趣的读者阅读原书。

DispatcherServlet 的实现是 SpringMVC 模块的重点内容，下面从初始化及调用流程来分析其实现原理。

## 快速体验

SpringMVC 快速体验（[源码Github地址](https://github.com/daigd/StudyDemo/tree/master/spring-web-mvc)），关键依赖如下，spring.version 为实际版本号：

```xml
<!--springMVC 依赖 begin-->
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-webmvc</artifactId>
	<version>${spring.version}</version>
</dependency>
<!--springMVC 依赖 end-->

<!-- 嵌入式Tomcat -->
<dependency>
	<groupId>org.apache.tomcat.embed</groupId>
	<artifactId>tomcat-embed-core</artifactId>
	<version>${tomcat.version}</version>
</dependency>
<dependency>
	<groupId>org.apache.tomcat.embed</groupId>
	<artifactId>tomcat-embed-jasper</artifactId>
	<version>${tomcat.version}</version>
</dependency>
```

web.xml 配置：

```xml
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
  <display-name>Archetype Created Web Application</display-name>
  
  <servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextClass</param-name>
      <param-value>org.springframework.web.context.support.AnnotationConfigWebApplicationContext</param-value>
    </init-param>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>com.dgd.WebApplication</param-value>
    </init-param>
    <load-on-startup>0</load-on-startup>
  </servlet>

  <servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/*</url-pattern>
  </servlet-mapping>
</web-app>
```

声明了 servlet-class 为 DispatcherServlet，上下文实现类为 AnnotationConfigWebApplicationContext，通过注解配置 web 容器环境，配置类为 WebApplication，内容如下：

```java
// 声明为配置类
@Configuration
// 扫描路径以当前类所在位置为根目录
@ComponentScan
// 启动 web mvc 功能
@EnableWebMvc
public class WebApplication
{
    public static void main(String[] args) throws Exception {
        Tomcat tomcat = new Tomcat();
        tomcat.setPort(Integer.getInteger("port", 8080));
        tomcat.getConnector();
        Context ctx = tomcat.addWebapp("", new File("src/main/webapp").getAbsolutePath());
        WebResourceRoot resources = new StandardRoot(ctx);
        resources.addPreResources(
                new DirResourceSet(resources, "/WEB-INF/classes", new File("target/classes").getAbsolutePath(), "/"));
        ctx.setResources(resources);
        tomcat.start();
        tomcat.getServer().await();
    }
}
```

控制器代码:

```	java
@RestController
public class HelloController
{
    private static final Logger LOGGER = LoggerFactory.getLogger(HelloController.class);

    @GetMapping("/")
    public String index() {
        Demo demo = new Demo();
        demo.setName("index");
        LOGGER.info("请求返回结果:{}", demo);
        return JSON.toJSONString(demo);
    }
    
    @GetMapping("/hello")
    public String hello() {
        Demo demo = new Demo();
        demo.setName("hello");
        LOGGER.info("请求返回结果:{}", demo);
        return JSON.toJSONString(demo);
    }
}
```

执行 WebApplication.main 方法，项目启动后，在浏览器输入 http://localhost:8080/，响应：

```json
{
  "name": "index"
}
```

成功走完一次 DispatcherServlet  调用流程，下面开始源码分析。

## 初始化流程

DispatcherServlet 实现了 Servlet 接口，先看下该接口的定义：

```java
public interface Servlet {
	// 初始化方法
    public void init(ServletConfig config) throws ServletException;
	// 获取配置信息
    public ServletConfig getServletConfig();
	// 处理请求方法
    public void service(ServletRequest req, ServletResponse res)
            throws ServletException, IOException;
    // 获取 Servlet 信息
    public String getServletInfo();
    // 销毁方法
    public void destroy();
}
```

 web 容器（比如本例中使用的 tomcat）加载 Servlet 及 Servlet 生命周期相关内容暂不讨论，从接口定义的方法中可以确定，DispatcherServlet 的初始化肯定也是走的 init()方法，处理请求也肯定是通过 service() 方法来进行处理，首先我们看下它是如何初始化的，在父类 GenericServlet 中找到 init(ServletConfig config) 方法的实现：

```java
public void init(ServletConfig config) throws ServletException {
    // config 传进来的内容包含 web.xml 配置的信息
    this.config = config;
    this.init();
}
// HttpServletBean.init() 进行了复写
// final 保证子类不能再对 init() 的逻辑进行改造
public final void init() throws ServletException {
	// Set bean properties from init parameters.
	// 属性初始化:载入init-param配置的属性
	PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
	if (!pvs.isEmpty()) {
		try {
			// 将当前对象(DispatcherServlet)包装成Spring可以处理的对象类型(BeanWrapper)
			BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
			ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
            // 注册自定义属性编辑器，一旦遇到 Resource 类型属性，便使用 ResourceEditor 来解析
			bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
			// 空实现,扩展预留
			initBeanWrapper(bw);
			// 属性注入
			bw.setPropertyValues(pvs, true);
		}
		catch (BeansException ex) {
			if (logger.isErrorEnabled()) {
				logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
			}
			throw ex;
		}
	}

	// Let subclasses do whatever initialization they like.
	// ServletBean 初始化
	initServletBean();
}
private static class ServletConfigPropertyValues extends MutablePropertyValues {
	public ServletConfigPropertyValues(ServletConfig config, Set<String> requiredProperties)
			throws ServletException {
		// 如果配置了必填参数,会对所需参数进行校验,参数缺失则抛出异常
		Set<String> missingProps = (!CollectionUtils.isEmpty(requiredProperties) ?
				new HashSet<>(requiredProperties) : null);

		Enumeration<String> paramNames = config.getInitParameterNames();
		while (paramNames.hasMoreElements()) {
			String property = paramNames.nextElement();
			Object value = config.getInitParameter(property);
			addPropertyValue(new PropertyValue(property, value));
			if (missingProps != null) {
				missingProps.remove(property);
			}
		}
		// Fail if we are still missing properties.
		if (!CollectionUtils.isEmpty(missingProps)) {
			throw new ServletException(
					"Initialization from ServletConfig for servlet '" + config.getServletName() +
					"' failed; the following required properties were missing: " +
					StringUtils.collectionToDelimitedString(missingProps, ", "));
		}
	}
}
```

- 加载 init-param 配置的属性；

  封装参数的时候，如果配置了必填参数，会对参数进行校验，参数缺失则抛出异常。

- 将当前 Servlet 转换成 BeanWrapper 实例；

- 注册自定义属性编辑器；

- 属性注入；

- ServletBean 初始化。

## ServletBean 初始化

initServletBean() 在 FrameworkServlet 类里进行了重写：

```java
protected final void initServletBean() throws ServletException {
	getServletContext().log("Initializing Spring " + getClass().getSimpleName() + " '" + getServletName() + "'");
	if (logger.isInfoEnabled()) {
		logger.info("Initializing Servlet '" + getServletName() + "'");
	}
	long startTime = System.currentTimeMillis();

	try {
		// 初始化逻辑入口：创建或刷新WebApplicationContext实例，并对servlet所使用的变量进行初始化
		this.webApplicationContext = initWebApplicationContext();
		// 空实现,扩展预留
		initFrameworkServlet();
	}
	catch (ServletException | RuntimeException ex) {
		logger.error("Context initialization failed", ex);
		throw ex;
	}
	if (logger.isDebugEnabled()) {
		String value = this.enableLoggingRequestDetails ?
				"shown which may lead to unsafe logging of potentially sensitive data" :
				"masked to prevent unsafe logging of potentially sensitive data";
		logger.debug("enableLoggingRequestDetails='" + this.enableLoggingRequestDetails +
				"': request parameters and headers will be " + value);
	}
	// 日志打印 Servlet 初始化消耗时间
	if (logger.isInfoEnabled()) {
		logger.info("Completed initialization in " + (System.currentTimeMillis() - startTime) + " ms");
	}
}
```

initServletBean() 重点逻辑就是对 WebApplicationContext 接口进行实例化：

```java
protected WebApplicationContext initWebApplicationContext() {
	WebApplicationContext rootContext =
			WebApplicationContextUtils.getWebApplicationContext(getServletContext());
	WebApplicationContext wac = null;

	if (this.webApplicationContext != null) {
		// A context instance was injected at construction time -> use it
		// context实例在构造函数中被注入，则直接使用
		wac = this.webApplicationContext;
		if (wac instanceof ConfigurableWebApplicationContext) {
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
			if (!cwac.isActive()) {
				// The context has not yet been refreshed -> provide services such as
				// setting the parent context, setting the application context id, etc
				if (cwac.getParent() == null) {
					// The context instance was injected without an explicit parent -> set
					// the root application context (if any; may be null) as the parent
					cwac.setParent(rootContext);
				}
				// 刷新上下文环境
				configureAndRefreshWebApplicationContext(cwac);
			}
		}
	}
	if (wac == null) {
		// No context instance was injected at construction time -> see if one
		// has been registered in the servlet context. If one exists, it is assumed
		// that the parent context (if any) has already been set and that the
		// user has performed any initialization such as setting the context id
		
		// context 没有在构造函数中注入，尝试在当前 servlet 去获取一个
		wac = findWebApplicationContext();
	}
	if (wac == null) {
		// No context instance is defined for this servlet -> create a local one
		// 找不到 context 实例，则直接创建一个
		wac = createWebApplicationContext(rootContext);
	}

	if (!this.refreshEventReceived) {
		// Either the context is not a ConfigurableApplicationContext with refresh
		// support or the context injected at construction time had already been
		// refreshed -> trigger initial onRefresh manually here.
		synchronized (this.onRefreshMonitor) {
			// 刷新 context
			onRefresh(wac);
		}
	}

	if (this.publishContext) {
		// Publish the context as a servlet context attribute.
		String attrName = getServletContextAttributeName();
        // 设置配置属性
		getServletContext().setAttribute(attrName, wac);
	}

	return wac;
}
```

- 初始化 WebApplicationContext 实例；
- 刷新 context，初始化各种资源处理器；
- 将当前 Servlet 及其属性设置到 ServletContext 中。

重点对 createWebApplicationContext(rootContext) 方法进行分析：

```java
protected WebApplicationContext createWebApplicationContext(@Nullable WebApplicationContext parent) {
	return createWebApplicationContext((ApplicationContext) parent);
}
protected WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent) {
	Class<?> contextClass = getContextClass();
	if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
		throw new ApplicationContextException(
				"Fatal initialization error in servlet with name '" + getServletName() +
				"': custom WebApplicationContext class [" + contextClass.getName() +
				"] is not of type ConfigurableWebApplicationContext");
	}
	// 利用反射创建 context 实例，对象类型为 ConfigurableWebApplicationContext
	ConfigurableWebApplicationContext wac =
			(ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);

	wac.setEnvironment(getEnvironment());
	wac.setParent(parent);
    // 设置配置文件路径
	String configLocation = getContextConfigLocation();
	if (configLocation != null) {
		wac.setConfigLocation(configLocation);
	}
	// 初始化Spring环境包括加载配置文件等
	configureAndRefreshWebApplicationContext(wac);
	return wac;
}

protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
	if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
		// The application context id is still set to its original default value
		// -> assign a more useful id based on available information
		// 设置 contextId
		if (this.contextId != null) {
			wac.setId(this.contextId);
		}
		else {
			// Generate default id...
			wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
					ObjectUtils.getDisplayString(getServletContext().getContextPath()) + '/' + getServletName());
		}
	}

	// 属性设置
	wac.setServletContext(getServletContext());
	wac.setServletConfig(getServletConfig());
	wac.setNamespace(getNamespace());
	wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));

	// The wac environment's #initPropertySources will be called in any case when the context
	// is refreshed; do it eagerly here to ensure servlet property sources are in place for
	// use in any post-processing or initialization that occurs below prior to #refresh
	ConfigurableEnvironment env = wac.getEnvironment();
	if (env instanceof ConfigurableWebEnvironment) {
		((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
	}

	// 扩展预留
	postProcessWebApplicationContext(wac);
	applyInitializers(wac);
	// 加载Spring容器及扩展实现
	wac.refresh();
}
```

对 WebApplicationContext 实例的过程最终会调用 AbstractApplicationContext.refresh() 方法，该部分内容可参看 [Spring 容器功能扩展](https://vigorous-wozniak-4b6bd2.netlify.app/%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80/java/spring%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0-%E5%AE%B9%E5%99%A8%E5%8A%9F%E8%83%BD%E6%89%A9%E5%B1%95/)，至此 Spring 容器已创建完毕，下面刷新 context，初始化 Web 场景下用到的一些处理器，实现代码在 DispatcherServlet.onRefresh(ApplicationContext context) 中：

```java
protected void onRefresh(ApplicationContext context) {
	initStrategies(context);
}
protected void initStrategies(ApplicationContext context) {
	// 初始化 MultipartResolver（主要用来处理文件上传）,beanName 为 multipartResolver
	initMultipartResolver(context);
	// 初始化 LocaleResolver（国际化配置）,beanName 为 localeResolver
	initLocaleResolver(context);
	// 初始化 ThemeResolver（涉及到网页主题的切换）,beanName 为 themeResolver
	initThemeResolver(context);
	// 初始化 HandlerMapping （负责请求的转发）
	initHandlerMappings(context);
	// 初始化 HandlerAdapter
	initHandlerAdapters(context);
	// 初始化 HandlerExceptionResolver
	initHandlerExceptionResolvers(context);
	// 初始化 RequestToViewNameTranslator
	initRequestToViewNameTranslator(context);
	// 初始化 ViewResolver
	initViewResolvers(context);
	// 初始化 FlashMapManager
	initFlashMapManager(context);
}
```

由于这个时候 Spring 已经创建完毕，所以各种处理器的初始化流程都是类似的：如果 Spring 容器有对应 Bean，则直接指向对应 Bean，如果没有，根据 DispatcherServlet.properties 配置的默认值来创建对应 Bean，如果配置文件也没有配置默认值，则将处理器设为 null。下面以 MultipartResolver 和 LocaleResolver 的初始化过程来展开分析，其它代码思路都是类似的，不再展开。

initMultipartResolver(context) 实现逻辑：

```java
private void initMultipartResolver(ApplicationContext context) {
	try {
		
		// MultipartResolver beanName 固定为 multipartResolver,
		this.multipartResolver = context.getBean(MULTIPART_RESOLVER_BEAN_NAME, MultipartResolver.class);
		if (logger.isTraceEnabled()) {
			logger.trace("Detected " + this.multipartResolver);
		}
		else if (logger.isDebugEnabled()) {
			logger.debug("Detected " + this.multipartResolver.getClass().getSimpleName());
		}
	}
	catch (NoSuchBeanDefinitionException ex) {
		// Default is no multipart resolver.
		// 找不到对应 Bean 则设置为 null
		this.multipartResolver = null;
		if (logger.isTraceEnabled()) {
			logger.trace("No MultipartResolver '" + MULTIPART_RESOLVER_BEAN_NAME + "' declared");
		}
	}
}
```

initLocaleResolver(context) 实现逻辑：

```java
private void initLocaleResolver(ApplicationContext context) {
	try {
		// 从 Spring 容器中获取,beanName 为 localeResolver
		this.localeResolver = context.getBean(LOCALE_RESOLVER_BEAN_NAME, LocaleResolver.class);
		if (logger.isTraceEnabled()) {
			logger.trace("Detected " + this.localeResolver);
		}
		else if (logger.isDebugEnabled()) {
			logger.debug("Detected " + this.localeResolver.getClass().getSimpleName());
		}
	}
	catch (NoSuchBeanDefinitionException ex) {
		// We need to use the default.
		// 如果用户没有定义,则使用默认的 LocaleResolver 实现
		this.localeResolver = getDefaultStrategy(context, LocaleResolver.class);
		if (logger.isTraceEnabled()) {
			logger.trace("No LocaleResolver '" + LOCALE_RESOLVER_BEAN_NAME +
					"': using default [" + this.localeResolver.getClass().getSimpleName() + "]");
		}
	}
}
```

加载默认实现：

```java
protected <T> T getDefaultStrategy(ApplicationContext context, Class<T> strategyInterface) {
	List<T> strategies = getDefaultStrategies(context, strategyInterface);
	if (strategies.size() != 1) {
		throw new BeanInitializationException(
				"DispatcherServlet needs exactly 1 strategy for interface [" + strategyInterface.getName() + "]");
	}
	return strategies.get(0);
}
protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) {
	String key = strategyInterface.getName();
	// 默认配置从 DispatcherServlet.properties 中加载
	String value = defaultStrategies.getProperty(key);
	if (value != null) {
		String[] classNames = StringUtils.commaDelimitedListToStringArray(value);
		List<T> strategies = new ArrayList<>(classNames.length);
		for (String className : classNames) {
			try {
                // 获取 Class 对象
				Class<?> clazz = ClassUtils.forName(className, DispatcherServlet.class.getClassLoader());
				// 根据 Class 创建对应 BeanDefinition
                Object strategy = createDefaultStrategy(context, clazz);
				strategies.add((T) strategy);
			}
			catch (ClassNotFoundException ex) {
				throw new BeanInitializationException(
						"Could not find DispatcherServlet's default strategy class [" + className +
						"] for interface [" + key + "]", ex);
			}
			catch (LinkageError err) {
				throw new BeanInitializationException(
						"Unresolvable class definition for DispatcherServlet's default strategy class [" +
						className + "] for interface [" + key + "]", err);
			}
		}
		return strategies;
	}
	else {
		return new LinkedList<>();
	}
}
```

createDefaultStrategy(context, clazz) 实现如下：

```java
protected Object createDefaultStrategy(ApplicationContext context, Class<?> clazz) {
	return context.getAutowireCapableBeanFactory().createBean(clazz);
}
// 创建 BeanDefinition
public <T> T createBean(Class<T> beanClass) throws BeansException {
	// Use prototype bean definition, to avoid registering bean as dependent bean.
	RootBeanDefinition bd = new RootBeanDefinition(beanClass);
	bd.setScope(SCOPE_PROTOTYPE);
	bd.allowCaching = ClassUtils.isCacheSafe(beanClass, getBeanClassLoader());
	return (T) createBean(beanClass.getName(), bd, null);
}
```

察看 DispatcherServlet.properties 内容，可知 LocaleResolver，ThemeResolver，HandlerMapping，HandlerAdapter，HandlerExceptionResolver，RequestToViewNameTranslator，ViewResolver 和 FlashMapManager 都提供了默认实现：

```properties
# Default implementation classes for DispatcherServlet's strategy interfaces.
# Used as fallback when no matching beans are found in the DispatcherServlet context.
# Not meant to be customized by application developers.

org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
	org.springframework.web.servlet.function.support.RouterFunctionMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
	org.springframework.web.servlet.function.support.HandlerFunctionAdapter


org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```

## DispatcherServlet 逻辑处理

Servlet 接口的 service(ServletRequest req, ServletResponse res) 方法 实现在 HttpServlet中，如下：

```java
public void service(ServletRequest req, ServletResponse res)
    throws ServletException, IOException
{
    HttpServletRequest  request;
    HttpServletResponse response;
    
    if (!(req instanceof HttpServletRequest &&
            res instanceof HttpServletResponse)) {
        throw new ServletException("non-HTTP request or response");
    }

    request = (HttpServletRequest) req;
    response = (HttpServletResponse) res;
	// 参数转化后，请求交由 service(HttpServletRequest req, HttpServletResponse resp) 方法处理
    service(request, response);
}
protected void service(HttpServletRequest req, HttpServletResponse resp)
    throws ServletException, IOException
{
    String method = req.getMethod();

    if (method.equals(METHOD_GET)) {
        long lastModified = getLastModified(req);
        if (lastModified == -1) {
            // servlet doesn't support if-modified-since, no reason
            // to go through further expensive logic
            doGet(req, resp);
        } else {
            long ifModifiedSince = req.getDateHeader(HEADER_IFMODSINCE);
            if (ifModifiedSince < lastModified) {
                // If the servlet mod time is later, call doGet()
                // Round down to the nearest second for a proper compare
                // A ifModifiedSince of -1 will always be less
                maybeSetLastModified(resp, lastModified);
                doGet(req, resp);
            } else {
                resp.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
            }
        }

    } else if (method.equals(METHOD_HEAD)) {
        long lastModified = getLastModified(req);
        maybeSetLastModified(resp, lastModified);
        doHead(req, resp);

    } else if (method.equals(METHOD_POST)) {
        doPost(req, resp);
        
    } else if (method.equals(METHOD_PUT)) {
        doPut(req, resp);
        
    } else if (method.equals(METHOD_DELETE)) {
        doDelete(req, resp);
        
    } else if (method.equals(METHOD_OPTIONS)) {
        doOptions(req,resp);
        
    } else if (method.equals(METHOD_TRACE)) {
        doTrace(req,resp);
        
    } else {
        //
        // Note that this means NO servlet supports whatever
        // method was requested, anywhere on this server.
        //
        String errMsg = lStrings.getString("http.method_not_implemented");
        Object[] errArgs = new Object[1];
        errArgs[0] = method;
        errMsg = MessageFormat.format(errMsg, errArgs);
        
        resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED, errMsg);
    }
}
```

可以看到，在HttpServlet.service(req,resp) 方法，根据方法名不同，分别交由对应的请求方法来处理，这里就针对 doGet(req, resp) 和 doPost(req,resp) 来展开分析。

doGet(req, resp)  和 doPost(req,resp) 方法实现逻辑在 FrameworkServlet 类中，实现如下：

```java
// final 禁止子类重写
protected final void doGet(HttpServletRequest request, HttpServletResponse response)
		throws ServletException, IOException {
	processRequest(request, response);
}
protected final void doPost(HttpServletRequest request, HttpServletResponse response)
		throws ServletException, IOException {
	processRequest(request, response);
}
```

可以看到，Get 请求 和Post 请求最后都是委托给 processRequest(request, response) 方法来处理：

```java
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
		throws ServletException, IOException {
	// 记录当前时间,用于计算当前web请求的处理时间
	long startTime = System.currentTimeMillis();
	Throwable failureCause = null;

	// 提取出当前线程的 LocaleContext、RequestAttributes 变量
	LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
	LocaleContext localeContext = buildLocaleContext(request);

	RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
	ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

	// 为当前请求创建 WebAsyncManager 实例，负责管理异步请求
	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
	asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());

	// 为当前请求初始化 LocaleContext、RequestAttributes 变量
	initContextHolders(request, localeContext, requestAttributes);

	try {
		// 请求处理逻辑 
		doService(request, response);
	}
	catch (ServletException | IOException ex) {
		failureCause = ex;
		throw ex;
	}
	catch (Throwable ex) {
		failureCause = ex;
		throw new NestedServletException("Request processing failed", ex);
	}

	finally {
		// 恢复 LocaleContext、RequestAttributes 变量
		resetContextHolders(request, previousLocaleContext, previousAttributes);
		if (requestAttributes != null) {
			requestAttributes.requestCompleted();
		}
		// 日志处理
		logResult(request, response, failureCause, asyncManager);
		// 发布事件通知
		publishRequestHandledEvent(request, response, startTime, failureCause);
	}
}
```

doService(request, response) 方法的实现在 DispatcherServlet 类中：

```java
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
	// 日志处理
	logRequest(request);

	// Keep a snapshot of the request attributes in case of an include,
	// to be able to restore the original attributes after the include.
	Map<String, Object> attributesSnapshot = null;
	// 判断请求中是否包含属性：javax.servlet.include.request_uri,包含则缓存属性快照
	if (WebUtils.isIncludeRequest(request)) {
		attributesSnapshot = new HashMap<>();
		Enumeration<?> attrNames = request.getAttributeNames();
		while (attrNames.hasMoreElements()) {
			String attrName = (String) attrNames.nextElement();
			if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
				attributesSnapshot.put(attrName, request.getAttribute(attrName));
			}
		}
	}

	// Make framework objects available to handlers and view objects.
	// 将辅助变量(webApplicationContext，localeResolver,themeResolver,themeSource)设置成 request 变量，
	// 方便后续过程使用
	request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
	request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
	request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
	request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

	if (this.flashMapManager != null) {
		FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
		if (inputFlashMap != null) {
			request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
		}
		request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
		request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
	}

	try {
		// 请求分发处理
		doDispatch(request, response);
	}
	finally {
		if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			// Restore the original attribute snapshot, in case of an include.
			// 恢复原始属性快照
			if (attributesSnapshot != null) {
				restoreAttributesAfterInclude(request, attributesSnapshot);
			}
		}
	}
}
```

请求分发处理实现：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	boolean multipartRequestParsed = false;
    // 获取异步管理器
	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

	try {
		ModelAndView mv = null;
		Exception dispatchException = null;

		try {
			// contentType 为 multipart 类型,将 request 转化为 MultipartHttpServletRequest 类型
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);

			// Determine handler for the current request.
			// 寻找对应的 handler 来处理请求
			mappedHandler = getHandler(processedRequest);
			if (mappedHandler == null) {
				// 没找到处理请求的 handler，则通过 response 反馈错误信息
				noHandlerFound(processedRequest, response);
				return;
			}

			// Determine handler adapter for the current request.
			// 根据当前的 handler 来确定对应的 handlerAdapter
			HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

			// Process last-modified header, if supported by the handler.
			String method = request.getMethod();
			boolean isGet = "GET".equals(method);
			if (isGet || "HEAD".equals(method)) {
				long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
				// 当前 handler 支持 last-modified 头处理
				if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
					return;
				}
			}

			// handler 拦截器 preHandle 方法调用
			if (!mappedHandler.applyPreHandle(processedRequest, response)) {
				return;
			}

			// Actually invoke the handler.
			// 处理请求,并返回视图
			mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

			if (asyncManager.isConcurrentHandlingStarted()) {
				return;
			}
			applyDefaultViewName(processedRequest, mv);
			// handler 拦截器 postHandle 方法调用
			mappedHandler.applyPostHandle(processedRequest, response, mv);
		}
		catch (Exception ex) {
			dispatchException = ex;
		}
		catch (Throwable err) {
			// As of 4.3, we're processing Errors thrown from handler methods as well,
			// making them available for @ExceptionHandler methods and other scenarios.
			dispatchException = new NestedServletException("Handler dispatch failed", err);
		}
		// 处理请求结果
		processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
	}
	catch (Exception ex) {
		// 请求处理完成之后触发
		triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
	}
	catch (Throwable err) {
		triggerAfterCompletion(processedRequest, response, mappedHandler,
				new NestedServletException("Handler processing failed", err));
	}
	finally {
		if (asyncManager.isConcurrentHandlingStarted()) {
			// Instead of postHandle and afterCompletion
			if (mappedHandler != null) {
				mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
			}
		}
		else {
			// Clean up any resources used by a multipart request.
			if (multipartRequestParsed) {
				// 针对 MultipartHttpServletRequest 请求做资源释放
				cleanupMultipart(processedRequest);
			}
		}
	}
}
```

### 获取异步管理器

WebAsyncUtils.getAsyncManager(request) 实现逻辑：

```java
public static WebAsyncManager getAsyncManager(ServletRequest servletRequest) {
	WebAsyncManager asyncManager = null;
	// 尝试从 request 属性中获取,如果没有再创建一个
	Object asyncManagerAttr = servletRequest.getAttribute(WEB_ASYNC_MANAGER_ATTRIBUTE);
	if (asyncManagerAttr instanceof WebAsyncManager) {
		asyncManager = (WebAsyncManager) asyncManagerAttr;
	}
	if (asyncManager == null) {
		asyncManager = new WebAsyncManager();
		servletRequest.setAttribute(WEB_ASYNC_MANAGER_ATTRIBUTE, asyncManager);
	}
	return asyncManager;
}
```

### multipart 请求处理

checkMultipart(request) 实现逻辑：

```java
protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
	if (this.multipartResolver != null && this.multipartResolver.isMultipart(request)) {
		if (WebUtils.getNativeRequest(request, MultipartHttpServletRequest.class) != null) {
			if (request.getDispatcherType().equals(DispatcherType.REQUEST)) {
				logger.trace("Request already resolved to MultipartHttpServletRequest, e.g. by MultipartFilter");
			}
		}
		else if (hasMultipartException(request)) {
			logger.debug("Multipart resolution previously failed for current request - " +
					"skipping re-resolution for undisturbed error rendering");
		}
		else {
			try {
				// 将 request 转化为 MultipartHttpServletRequest 类型
				return this.multipartResolver.resolveMultipart(request);
			}
			catch (MultipartException ex) {
				if (request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) != null) {
					logger.debug("Multipart resolution failed for error dispatch", ex);
					// Keep processing error dispatch with regular request handle below
				}
				else {
					throw ex;
				}
			}
		}
	}
	// If not returned before: return original request.
	return request;
}
```

### 根据 request 信息获取对应的 handler

getHandler(processedRequest) 实现逻辑：

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	if (this.handlerMappings != null) {
		for (HandlerMapping mapping : this.handlerMappings) {
			HandlerExecutionChain handler = mapping.getHandler(request);
			if (handler != null) {
				return handler;
			}
		}
	}
	return null;
}
```

HandlerMapping.getHandler(request) 的实现在 AbstractHandlerMapping 类中，

```java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	// 根据请求获取对应的 handler
	Object handler = getHandlerInternal(request);
	if (handler == null) {
		// 找不到 handler,使用默认的 handler
		handler = getDefaultHandler();
	}
	// handler 为空,直接返回
	if (handler == null) {
		return null;
	}
	// Bean name or resolved handler?
	if (handler instanceof String) {
		// 如果 handler 为字符串，尝试从 Spring 容器获取对应 bean
		String handlerName = (String) handler;
		handler = obtainApplicationContext().getBean(handlerName);
	}
	// 实际执行逻辑
	HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

	if (logger.isTraceEnabled()) {
		logger.trace("Mapped to " + handler);
	}
	else if (logger.isDebugEnabled() && !request.getDispatcherType().equals(DispatcherType.ASYNC)) {
		logger.debug("Mapped to " + executionChain.getHandler());
	}
	// 针对跨域请求处理
	if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
		CorsConfiguration config = (this.corsConfigurationSource != null ? this.corsConfigurationSource.getCorsConfiguration(request) : null);
		CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
		config = (config != null ? config.combine(handlerConfig) : handlerConfig);
		executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
	}
	return executionChain;
}
```

### 没找到 handler 的异常处理

noHandlerFound(processedRequest, response) 实现逻辑：

```java
protected void noHandlerFound(HttpServletRequest request, HttpServletResponse response) throws Exception {
	if (pageNotFoundLogger.isWarnEnabled()) {
		pageNotFoundLogger.warn("No mapping for " + request.getMethod() + " " + getRequestUri(request));
	}
	if (this.throwExceptionIfNoHandlerFound) {
		throw new NoHandlerFoundException(request.getMethod(), getRequestUri(request),
				new ServletServerHttpRequest(request).getHeaders());
	}
	else {
		// 设置 response 响应码为404 
		response.sendError(HttpServletResponse.SC_NOT_FOUND);
	}
}
```

### 根据 handler 获取对应的 handlerAdapter

getHandlerAdapter(mappedHandler.getHandler()) 代码如下：

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
	if (this.handlerAdapters != null) {
		for (HandlerAdapter adapter : this.handlerAdapters) {
			if (adapter.supports(handler)) {
				return adapter;
			}
		}
	}
	throw new ServletException("No adapter for handler [" + handler +
			"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

### last-modified 缓存处理

如果是 Get 或 Head 协议请求，且服务器在指定时间内容没有修改过，响应状态码返回 304，只有响应头，内容为空，这样就节省了网络带宽。

### handler 拦截器 preHandle 方法调用

### 处理请求并返回视图

代码入口 ha.handle(processedRequest, response, mappedHandler.getHandler())，实现逻辑在 AbstractHandlerMethodAdapter中：

```java
public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
		throws Exception {
	return handleInternal(request, response, (HandlerMethod) handler);
}
```

handleInternal(request, response, (HandlerMethod) handler) 实现在 RequestMappingHandlerAdapter 类中：

```java
protected ModelAndView handleInternal(HttpServletRequest request,
		HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

	ModelAndView mav;
	// 检查该请求是否支持处理及是否需要会话
	checkRequest(request);

	// Execute invokeHandlerMethod in synchronized block if required.
	// 执行同步请求
	if (this.synchronizeOnSession) {
		HttpSession session = request.getSession(false);
		if (session != null) {
			Object mutex = WebUtils.getSessionMutex(session);
			synchronized (mutex) {
				mav = invokeHandlerMethod(request, response, handlerMethod);
			}
		}
		else {
			// No HttpSession available -> no mutex necessary
			mav = invokeHandlerMethod(request, response, handlerMethod);
		}
	}
	else {
		// No synchronization on session demanded at all...
		// GET 请求会直接进来这里
		mav = invokeHandlerMethod(request, response, handlerMethod);
	}

	if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
		if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
			applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
		}
		else {
			prepareResponse(response);
		}
	}

	return mav;
}
```

重点关注 invokeHandlerMethod(request, response, handlerMethod) 方法：

```java

```



