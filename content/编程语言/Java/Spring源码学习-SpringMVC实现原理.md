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
```



