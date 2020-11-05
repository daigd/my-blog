---
title: "Spring源码学习-容器功能扩展"
date: 2020-10-23T09:47:12+08:00
tags: ["Spring", "源码学习","IOC","容器功能扩展"]
draft: false
---

> Spring 版本：5.2.6.RELEASE，笔者的Spring源码学习笔记参考 郝佳 编著的《Spring源码深度解析》，推荐感兴趣的读者阅读原书。

[源码Github地址](https://github.com/daigd/StudyDemo/tree/master/spring-source-learning)。

ApplicationContext 和 BeanFactory 都是用于加载 Bean 的，但是前者提供了更多的扩展功能，实际应用中我们也是使用 ApplicationContext，下面我们从代码角度来看下其是如何对容器功能进行扩展的。

要使用 ApplicationContext 先引入对应依赖，如下：

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.2.6.RELEASE</version>
</dependency>
```

测试代码入口：

```java
public class MyApplicationContextTests
{
    @Test
    @DisplayName("容器实现-ApplicationContext-xml方式")
    void xmlApplicationContextTest()
    {
        ApplicationContext bf = new ClassPathXmlApplicationContext("MyBeanFactoryTests.xml");
        MyTestBean bean = (MyTestBean) bf.getBean("myTestBean");
        Assertions.assertEquals("myTest", bean.getName());
        System.out.println("myTest:" + bean.getName());
    }
}
```

以 ClassPathXmlApplicationContext 的初始化作为切入点，代码逻辑如下：

```java
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
    // refresh 参数为 true
	this(new String[] {configLocation}, true, null);
}

public ClassPathXmlApplicationContext(
		String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
		throws BeansException {

	super(parent);
	setConfigLocations(configLocations);
	if (refresh) {
        // 资源解析及加载
		refresh();
	}
}
```

ClassPathXmlApplicationContext 中可以将资源路径以数组的形式传入，之后对资源进行解析和加载，逻辑都在 refresh() 方法中实现。

## 设置配置路径

```java
setConfigLocations(configLocations);

public void setConfigLocations(@Nullable String... locations) {
	if (locations != null) {
		Assert.noNullElements(locations, "Config locations must not be null");
		this.configLocations = new String[locations.length];
		for (int i = 0; i < locations.length; i++) {
            // 设置加载资源路径
			this.configLocations[i] = resolvePath(locations[i]).trim();
		}
	}
	else {
		this.configLocations = null;
	}
}

```

设置了资源加载路径后，功能扩展逻辑全封装在了 refresh() 方法中，看下具体实现：

```java
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
        // 刷新上下文环境准备
		prepareRefresh();

		// Tell the subclass to refresh the internal bean factory.
        // 初始化 BeanFactory
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// Prepare the bean factory for use in this context.
        // 对 BeanFactory 进行功能填充
		prepareBeanFactory(beanFactory);

		try {
			// Allows post-processing of the bean factory in context subclasses.
            // 子类覆盖方法做额外处理
			postProcessBeanFactory(beanFactory);

			// Invoke factory processors registered as beans in the context.
            // 应用 BeanFactory 后置处理器
			invokeBeanFactoryPostProcessors(beanFactory);

			// Register bean processors that intercept bean creation.
			// 注册 Bean 后置处理器
            registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
            // 初始化 Message 源
			initMessageSource();

			// Initialize event multicaster for this context.
            // 初始化应用消息广播器
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
            // 给子类来刷新其它 Bean
			onRefresh();

			// Check for listener beans and register them.
            // 查找监听器 Bean,注册到消息广播中
			registerListeners();

			// Instantiate all remaining (non-lazy-init) singletons.
            // 初始化剩余的非惰性单例 Bean
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
            // 完成刷新过程，发布相应事件通知
			finishRefresh();
		}

		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();

			// Reset 'active' flag.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}

		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			resetCommonCaches();
		}
	}
}

```

入口方法主要封装了以下的逻辑：

- 刷新上下文环境准备；
- 初始化 BeanFactory；
- 对 BeanFactory 进行功能填充；
- ApplicationContext 子类覆盖方法处理；
- 应用 BeanFactory 后置处理器；
- 注册 Bean 后置处理器；
- 初始化 Message 源；
- 初始化应用消息广播器；
- 刷新其它 Bean；
- 查找监听器 Bean,注册到消息广播中；
- 初始化剩余的非惰性单例 Bean；
- 完成刷新过程，发布相应事件通知。

本篇文章先对刷新上下文环境准备及初始化 BeanFactory 的源代码做下分析。

## 环境准备

prepareRefresh() 方法主要是做些准备工作，例如对系统属性及环境变量的初始化及校验。

```java
protected void prepareRefresh() {
	// Switch to active.
    // 给 context 声明启动时间及激活状态
	this.startupDate = System.currentTimeMillis();
	this.closed.set(false);
	this.active.set(true);

	if (logger.isDebugEnabled()) {
		if (logger.isTraceEnabled()) {
			logger.trace("Refreshing " + this);
		}
		else {
			logger.debug("Refreshing " + getDisplayName());
		}
	}

	// Initialize any placeholder property sources in the context environment.
    // 进行个性化属性设置及处理，该方法为空实现，所以是留给子类实现扩展用的
	initPropertySources();

	// Validate that all properties marked as required are resolvable:
	// see ConfigurablePropertyResolver#setRequiredProperties
    // 对必传属性进行校验
	getEnvironment().validateRequiredProperties();

	// Store pre-refresh ApplicationListeners...
    // 初始化应用监听器
	if (this.earlyApplicationListeners == null) {
		this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
	}
	else {
		// Reset local application listeners to pre-refresh state.
		this.applicationListeners.clear();
		this.applicationListeners.addAll(this.earlyApplicationListeners);
	}

	// Allow for the collection of early ApplicationEvents,
	// to be published once the multicaster is available...
	this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

## 加载 BeanFactory

ApplicationContext 是在 BeanFactory 基础上进行了功能扩展，经过 obtainFreshBeanFactory() 方法的调用后，ApplicationContext 就拥有了 BeanFactory 全部功能，我们来看下其具体实现：

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	// 初始化 BeanFactory,并读取 xml 配置信息，交将得到的 BeanFactory 记录在当前实体的属性中
    refreshBeanFactory();
    // 返回当前实体的 BeanFactory 属性
	return getBeanFactory();
}
// AbstractRefreshableApplicationContext类中
protected final void refreshBeanFactory() throws BeansException {
    // 如果 BeanFactory 不为空,把对应单例相关缓存全部清除，并把 BeanFactory 设为 null
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	try {
        // 创建 BeanFactory
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		// 设置序列化Id
        beanFactory.setSerializationId(getId());
        // 定制BeanFactory，设置相关属性
		customizeBeanFactory(beanFactory);
        // 加载 BeanDefinition
		loadBeanDefinitions(beanFactory);
        // 创建好的BeanFactory设为当前实例的BeanFactory
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory;
		}
	}
	catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}
```

上面代码主要做了下面几步工作：

- BeanFactory存在校验，如果存在，先把对应单例缓存清除并把当前BeanFactory设为null；
- 创建 BeanFactory 实例对象，类型为DefaultListableBeanFactory；
- 设置序列化ID；
- 定制 BeanFactory，设置是否允许 BeanDefinition 重写及是否允许循环依赖；
- 加载 BeanDefinition；
- 将创建好的 BeanFactory 设为当前实例的 BeanFactory。

我们把重点放在 loadBeanDefinitions(beanFactory) 中，从 XML 加载 Bean 信息的逻辑都封装在了这个方法中。

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
	// Create a new XmlBeanDefinitionReader for the given BeanFactory.
	// 给 beanFactory 创建 XML解析器
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

	// Configure the bean definition reader with this context's
	// resource loading environment.
    // 设置相关属性
	beanDefinitionReader.setEnvironment(this.getEnvironment());
	beanDefinitionReader.setResourceLoader(this);
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

	// Allow a subclass to provide custom initialization of the reader,
	// then proceed with actually loading the bean definitions.
	// 设置校验标识为 true ，允许子类重写该方法，实现定制的属性处理
    initBeanDefinitionReader(beanDefinitionReader);
    // 真正加载 BeanDefinition 的入口
	loadBeanDefinitions(beanDefinitionReader);
}
```

真正加载 BeanDefinition 的工作委托给了 loadBeanDefinitions(beanDefinitionReader)，其实现如下：

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    // 根据资源路径去加载 BeanDefinition
	Resource[] configResources = getConfigResources();
	if (configResources != null) {
		reader.loadBeanDefinitions(configResources);
	}
	String[] configLocations = getConfigLocations();
	if (configLocations != null) {
		reader.loadBeanDefinitions(configLocations);
	}
}

// AbstractBeanDefinitionReader 类
public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
	Assert.notNull(locations, "Location array must not be null");
	int count = 0;
	for (String location : locations) {
		count += loadBeanDefinitions(location);
	}
	return count;
}

public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
	return loadBeanDefinitions(location, null);
}

public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
	ResourceLoader resourceLoader = getResourceLoader();
	if (resourceLoader == null) {
		throw new BeanDefinitionStoreException(
				"Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
	}

	if (resourceLoader instanceof ResourcePatternResolver) {
		// Resource pattern matching available.
		try {
			Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
            // 真正加载 BeanDefinition 的代码
			int count = loadBeanDefinitions(resources);
			if (actualResources != null) {
				Collections.addAll(actualResources, resources);
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
			}
			return count;
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"Could not resolve bean definition resource pattern [" + location + "]", ex);
		}
	}
	else {
		// Can only load single resources by absolute URL.
		Resource resource = resourceLoader.getResource(location);
		// 真正加载 BeanDefinition 的代码
        int count = loadBeanDefinitions(resource);
		if (actualResources != null) {
			actualResources.add(resource);
		}
		if (logger.isTraceEnabled()) {
			logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
		}
		return count;
	}
}
```

代码绕来绕去后，最后都是调用了 BeanDefinitionReader 接口的 loadBeanDefinitions(Resource resource) 方法，由于我们一开始创建的 Reader 是XmlBeanDefinitionReader，故最终也是委托了 XmlBeanDefinitionReader 来实现，其代码如下：

```java
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
	return loadBeanDefinitions(new EncodedResource(resource));
}
```

又来到了熟悉的代码入口，这里就跟 BeanFactory 加载 XML 配置信息的口子对上了，详情可参考笔者之前的学习笔记[spring源码学习-加载解析及注册xml配置信息](https://vigorous-wozniak-4b6bd2.netlify.app/%E7%BC%96%E7%A8%8B%E8%AF%AD%E8%A8%80/java/spring%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0-%E5%8A%A0%E8%BD%BD%E8%A7%A3%E6%9E%90%E5%8F%8A%E6%B3%A8%E5%86%8Cxml%E9%85%8D%E7%BD%AE%E4%BF%A1%E6%81%AF/)，这里不再赘述。

经过此步骤后，DefaultListableBeanFactory 类型的变量 beanFactory 已经包含了所有解析好的 BeanDefinition 配置。

## 对 BeanFactory 进行功能填充

接下来我们看下 Spring 如何对 BeanFactory 进行功能填充，代码入口为 prepareBeanFactory(beanFactory)，实现如下：

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	// Tell the internal bean factory to use the context's class loader etc.
    // 设置 beanFactory 的 classLoader 为当前 context 的 classLoader
	beanFactory.setBeanClassLoader(getClassLoader());
    // 设置SPEL表达式语言处理器
	beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
	// 增加默认的 PropertyEditorRegistrar，主要是针对 bean 的属性等设置管理的一个工具
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

	// Configure the bean factory with context callbacks.
    // 配置上下文回调处理器
	beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    // 设置忽略自动装配的接口
	beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
	beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
	beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
	beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

	// BeanFactory interface not registered as resolvable type in a plain factory.
	// MessageSource registered (and found for autowiring) as a bean.
	// 注册一些固定依赖
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
	beanFactory.registerResolvableDependency(ResourceLoader.class, this);
	beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
	beanFactory.registerResolvableDependency(ApplicationContext.class, this);

	// Register early post-processor for detecting inner beans as ApplicationListeners.
    // 注册 Bean 后置处理器，以将一些 Bean 转成 ApplicationListener
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

	// Detect a LoadTimeWeaver and prepare for weaving, if found.
	// 增加对 AspectJ 的支持
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		// Set a temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}

	// Register default environment beans.
    // 注册默认的系统环境 Bean
	if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
	}
}
```

察看上述代码，我们大体知道了对 BeanFactory 进行了以下功能的填充：

- 增加对 SPEL 语言的支持；
- 增加对属性编辑器的支持；
- 设置忽略自动装配的接口；
- 注册一些固定依赖的属性；
- 增加对 AspectJ 的支持；
- 将相关环境变量及属性以单例注册。

## 激活注册的 BeanFactoryPostProcessor

接着我们看下 postProcessBeanFactory(beanFactory) ，这是个空实现，主要是留给子类重写使用，重点看下 invokeBeanFactoryPostProcessors(beanFactory)，代码如下:

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
 	// 激活注册的后置处理器
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

	// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
	// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
	if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}
}

public static void invokeBeanFactoryPostProcessors(
		ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

	// Invoke BeanDefinitionRegistryPostProcessors first, if any.
	Set<String> processedBeans = new HashSet<>();

    // 对 BeanDefinitionRegistry 类型的处理
	if (beanFactory instanceof BeanDefinitionRegistry) {
		BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
		List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
		List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

        // 硬编码注册后处理器
		for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
			if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
				BeanDefinitionRegistryPostProcessor registryProcessor =
						(BeanDefinitionRegistryPostProcessor) postProcessor;
                // 对 BeanDefinitionRegistryPostProcessor 做一些额外处理
				registryProcessor.postProcessBeanDefinitionRegistry(registry);
				registryProcessors.add(registryProcessor);
			}
			else {
                // 记录常规 BeanFactoryPostProcessor
				regularPostProcessors.add(postProcessor);
			}
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
		// Separate between BeanDefinitionRegistryPostProcessors that implement
		// PriorityOrdered, Ordered, and the rest.
		List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

		// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
		for (String ppName : postProcessorNames) {
            // 从 beanFactory 挑选出实现了 PriorityOrdered 接口的 BeanDefinitionRegistryPostProcessor 实例
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
				processedBeans.add(ppName);
			}
		}
        // 对 BeanDefinitionRegistryPostProcessor 实例 进行排序
        // 即按 PriorityOrdered 排序
		sortPostProcessors(currentRegistryProcessors, beanFactory);
		registryProcessors.addAll(currentRegistryProcessors);
        // 调用处理方法：在 Bean 实例化前添加更多的 bean 定义
		invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
		currentRegistryProcessors.clear();

		// Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
        // 按 Ordered 排序
		postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
		for (String ppName : postProcessorNames) {
			if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
				currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
				processedBeans.add(ppName);
			}
		}
		sortPostProcessors(currentRegistryProcessors, beanFactory);
		registryProcessors.addAll(currentRegistryProcessors);
        // 调用处理方法
		invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
		currentRegistryProcessors.clear();

		// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
		boolean reiterate = true;
		while (reiterate) {
			reiterate = false;
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (!processedBeans.contains(ppName)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
					reiterate = true;
				}
			}
            // 没有实现PriorityOrdered、Ordered的，按默认规则排序
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
            // 调用处理方法
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();
		}

		// Now, invoke the postProcessBeanFactory callback of all processors handled so far.
        // 调用后置处理器方法
		invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
	}

	else {
		// Invoke factory processors registered with the context instance.
		invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
	}

	// Do not initialize FactoryBeans here: We need to leave all regular beans
	// uninitialized to let the bean factory post-processors apply to them!
	String[] postProcessorNames =
			beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

	// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
	// Ordered, and the rest.
	List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
	List<String> orderedPostProcessorNames = new ArrayList<>();
	List<String> nonOrderedPostProcessorNames = new ArrayList<>();
	for (String ppName : postProcessorNames) {
		if (processedBeans.contains(ppName)) {
			// skip - already processed in first phase above
		}
		else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
			priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
		}
		else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
			orderedPostProcessorNames.add(ppName);
		}
		else {
			nonOrderedPostProcessorNames.add(ppName);
		}
	}

	// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
	sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
	invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

	// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
	List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
	for (String postProcessorName : orderedPostProcessorNames) {
		orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
	}
	sortPostProcessors(orderedPostProcessors, beanFactory);
	invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

	// Finally, invoke all other BeanFactoryPostProcessors.
	List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
	for (String postProcessorName : nonOrderedPostProcessorNames) {
		nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
	}
	invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

	// Clear cached merged bean definitions since the post-processors might have
	// modified the original metadata, e.g. replacing placeholders in values...
	beanFactory.clearMetadataCache();
}

```

从代码中可以看到，对 BeanFactoryPostProcessor 的处理分两种情况，一种是针对 BeanDefinitionRegistryPostProcessor 类的特殊处理，另一种是针对普通 BeanFactoryPostProcessor 的处理，之后操作都是类似的，首先排序，然后添加额外的 bean 定义，最后调用 Processor 对应的处理方法。

## 注册 BeanPostProcessor

代码入口 registerBeanPostProcessors(beanFactory)，具体实现如下：

```java
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
	PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}

public static void registerBeanPostProcessors(
		ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

	String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

	// Register BeanPostProcessorChecker that logs an info message when
	// a bean is created during BeanPostProcessor instantiation, i.e. when
	// a bean is not eligible for getting processed by all BeanPostProcessors.
	int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
	// 添加一个自定义的 BeanPostProcessor
	beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

	// Separate between BeanPostProcessors that implement PriorityOrdered,
	// Ordered, and the rest.
	List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
	List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
	List<String> orderedPostProcessorNames = new ArrayList<>();
	List<String> nonOrderedPostProcessorNames = new ArrayList<>();
	
	// 按 PriorityOrdered、Ordered及其它类型对 BeanPostProcessor 进行分组
	for (String ppName : postProcessorNames) {
		if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			priorityOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
			orderedPostProcessorNames.add(ppName);
		}
		else {
			nonOrderedPostProcessorNames.add(ppName);
		}
	}

	// First, register the BeanPostProcessors that implement PriorityOrdered.
	// 排序，然后注册 BeanPostProcessor
	sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

	// Next, register the BeanPostProcessors that implement Ordered.
	List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
	for (String ppName : orderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		orderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	sortPostProcessors(orderedPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, orderedPostProcessors);

	// Now, register all regular BeanPostProcessors.
	List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
	for (String ppName : nonOrderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		nonOrderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

	// Finally, re-register all internal BeanPostProcessors.
	sortPostProcessors(internalPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, internalPostProcessors);

	// Re-register post-processor for detecting inner beans as ApplicationListeners,
	// moving it to the end of the processor chain (for picking up proxies etc).
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```

## 初始化消息资源

Spring 定义了访问国际化信息的 MessageSource 接口，并提供了几个实现类。下面看下 Spring 是如何为 ApplicationContext 添加访问国际化信息的功能的，代码入口 initMessageSource()，实现如下：

```java
protected void initMessageSource() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	// beanName 固定为：messageSource，如果容器已注册该名字的 bean，将其设置为 context 的 messageSource
	if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
		this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
		// Make MessageSource aware of parent MessageSource.
		if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
			HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
			if (hms.getParentMessageSource() == null) {
				// Only set parent context as parent MessageSource if no parent MessageSource
				// registered already.
				hms.setParentMessageSource(getInternalParentMessageSource());
			}
		}
		if (logger.isTraceEnabled()) {
			logger.trace("Using MessageSource [" + this.messageSource + "]");
		}
	}
	else {
		// Use empty MessageSource to be able to accept getMessage calls.
		// 如果容器中未有 beanName = messageSource 的 bean,使用MessageSource的代理类来初始化 messageSource 
		DelegatingMessageSource dms = new DelegatingMessageSource();
		dms.setParentMessageSource(getInternalParentMessageSource());
		this.messageSource = dms;
		beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
		if (logger.isTraceEnabled()) {
			logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
		}
	}
}
```

## 初始化容器事件传播器（ApplicationEventMulticaster）

Spring 是利用观察者模式来实现事件传播的，其初始化代码入口是 initApplicationEventMulticaster()，实现如下：

```java
protected void initApplicationEventMulticaster() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	// 如果已注册了自定义的事件广播器，那使用该广播器
	if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
		this.applicationEventMulticaster =
				beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
		if (logger.isTraceEnabled()) {
			logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
		}
	}
	else {
		// 默认使用 SimpleApplicationEventMulticaster 当容器事件广播器
		this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
		beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
		if (logger.isTraceEnabled()) {
			logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
					"[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
		}
	}
}
```

顺便我们看下监听器注册逻辑，代码入口 registerListeners()，实现：

```java
protected void registerListeners() {
	// Register statically specified listeners first.
	// 硬编码注册的监听器，注册到广播器中
	for (ApplicationListener<?> listener : getApplicationListeners()) {
		getApplicationEventMulticaster().addApplicationListener(listener);
	}

	// Do not initialize FactoryBeans here: We need to leave all regular beans
	// uninitialized to let post-processors apply to them!
	// 托管给Spring容器的监听器，注册到广播器中
	String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
	for (String listenerBeanName : listenerBeanNames) {
		getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
	}

	// Publish early application events now that we finally have a multicaster...
	Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
	this.earlyApplicationEvents = null;
	if (earlyEventsToProcess != null) {
		// 事件广播器已注册，可以发布一些需要提前发布的容器事件
		for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
			getApplicationEventMulticaster().multicastEvent(earlyEvent);
		}
	}
}
```

SimpleApplicationEventMulticaster 调用监听事件处理的逻辑如下：

```java
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
	ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
	Executor executor = getTaskExecutor();
	// 获取注册的事件监听者，执行处理方法
	for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
		if (executor != null) {
			executor.execute(() -> invokeListener(listener, event));
		}
		else {
			invokeListener(listener, event);
		}
	}
}
```

## 初始化非延迟加载单例

代码入口 finishBeanFactoryInitialization(beanFactory)，实现如下：

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
	// Initialize conversion service for this context.
	// 设置conversionService
	if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
			beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
		beanFactory.setConversionService(
				beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
	}

	// Register a default embedded value resolver if no bean post-processor
	// (such as a PropertyPlaceholderConfigurer bean) registered any before:
	// at this point, primarily for resolution in annotation attribute values.
	// 如果未注册值解析器，注册默认的值解析器
	if (!beanFactory.hasEmbeddedValueResolver()) {
		beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
	}

	// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
	// 提前实例化 LoadTimeWeaverAware 类型 bean
	String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
	for (String weaverAwareName : weaverAwareNames) {
		getBean(weaverAwareName);
	}

	// Stop using the temporary ClassLoader for type matching.
	beanFactory.setTempClassLoader(null);

	// Allow for caching all bean definition metadata, not expecting further changes.
	// 冻结全部 beanDefinition,即 bean 配置信息不会再改变
	beanFactory.freezeConfiguration();

	// Instantiate all remaining (non-lazy-init) singletons.
	// 实例化非延迟加载的 bean，最终也是调用 AbstractBeanFactory 的getBean()方法
	beanFactory.preInstantiateSingletons();
}
```

## 完成容器刷新

最后一步，finishRefresh()，实现逻辑：

```java
protected void finishRefresh() {
	// Clear context-level resource caches (such as ASM metadata from scanning).
	// 清除 context 级别的资源缓存
	clearResourceCaches();

	// Initialize lifecycle processor for this context.
	// 初始化 lifecycle 处理器
	initLifecycleProcessor();

	// Propagate refresh to lifecycle processor first.
	// 刷新 lifecycle 处理器
	getLifecycleProcessor().onRefresh();

	// Publish the final event.
	// 发布 context 刷新事件,以让对应监听器做相应处理
	publishEvent(new ContextRefreshedEvent(this));

	// Participate in LiveBeansView MBean, if active.
	// 如果配置了 spring.liveBeansView.mbeanDomain 属性，将当前容器名称注册到 MBeanServer 中
	LiveBeansView.registerApplicationContext(this);
}
```

- onRefresh():

  ```java
  public void onRefresh() {
  	// 启动所有实现了 Lifecycle 接口的 bean
  	startBeans(true);
  	this.running = true;
  }
  private void startBeans(boolean autoStartupOnly) {
  	// 获取实现了 Lifecycle 接口的bean，key-beanName,value-bean 
  	Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
  	Map<Integer, LifecycleGroup> phases = new HashMap<>();
  	lifecycleBeans.forEach((beanName, bean) -> {
  		if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
  			int phase = getPhase(bean);
  			LifecycleGroup group = phases.get(phase);
  			if (group == null) {
  				group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
  				phases.put(phase, group);
  			}
  			group.add(beanName, bean);
  		}
  	});
  	if (!phases.isEmpty()) {
  		List<Integer> keys = new ArrayList<>(phases.keySet());
  		Collections.sort(keys);
  		for (Integer key : keys) {
  			// 执行 Lifecycle start()方法
  			phases.get(key).start();
  		}
  	}
  }
  ```

- publishEvent:

  ```java
  public void publishEvent(ApplicationEvent event) {
  	// 发布容器启动事件给全部监听器，对应监听器可以做对应处理
  	publishEvent(event, null);
  }
  ```

  