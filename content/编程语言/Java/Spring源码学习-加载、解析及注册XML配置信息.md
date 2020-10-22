---
title: "Spring源码学习-加载、解析及注册XML配置信息"
date: 2020-20-10T16:51:12+08:00
tags: ["Spring", "源码学习","IOC"]
draft: false
---

> Spring 版本：5.2.6.RELEASE，笔者的Spring源码学习笔记参考 郝佳 编著的《Spring源码深度解析》，推荐感兴趣的读者阅读原书。

对 `Spring` 容器的使用，我们看一个简单示例，如下所示：

代码示例

Bean定义

```java
public class MyTestBean
{
    private String name = "myTest";

    public String getName()
    {
        return name;
    }

    public void setName(String name)
    {
        this.name = name;
    }
}
```

配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN 2.0//EN" "https://www.springframework.org/dtd/spring-beans-2.0.dtd">

<beans>
	<bean id="myTestBean" class="com.dgd.spring.beans.factory.MyTestBean" scope="prototype">
	</bean>
</beans>	
```

测试代码入口

```java
public class MyBeanFactoryBeanTests
{
    @Test
    @DisplayName("容器实现-xml方式")
    void xmlBeanFactoryTest()
    {
        BeanFactory bf = new XmlBeanFactory(new ClassPathResource("MyBeanFactoryTests.xml"));
        MyTestBean bean = (MyTestBean) bf.getBean("myTestBean");
        Assertions.assertEquals("myTest", bean.getName());
        System.out.println("myTest:" + bean.getName());
    }
}
```

## 核心类图

测试代码逻辑：创建一个能从xml文件中获取Bean的工厂类，然后通过Bean的名称获取指定Bean，从代码中可以发现，`XmlBeanFactory`是逻辑切入点，我们来看下它的上层核心类图，如下所示：

![](/java/img/XmlBeanFactory.png)

从左往右看，`XmlBeanFactory`的顶级父类有三，分别是`AliasRegistry`，`SingletonBeanRegistry`，`BeanFactory`，它们的作用分别是：

- AliasRegistry：定义对别名（ alias ）的注册、移除、判断别名、获取别名操作；
- SingletonBeanRegistry：定义对单例的注册和获取相关方法；
- BeanFactory：定义获取 Bean 及判断 Bean 的属性如是否是单例，类型是否匹配等相关方法。

从上往下看，重要的类有`BeanDefinitionRegistry`，`AbstractBeanFactroy`，`ConfigurableListableBeanFactory`：

- BeanDefinitionRegistry：继承自 `AliasRegistry`，添加了对 `BeanDefinition`（`BeanDefinition`提供了描述`Bean`定义的方法） 的各种操作；
- AbstractBeanFactroy：直接继承的父类`FactoryBeanRegistrySupport`,实现接口`ConfigurableBeanFactory`，综合了二者功能：
  - FactoryBeanRegistrySupport：继承`DafaultSingletonBeanRegistry`,`DafaultSingletonBeanRegistry` 提供 `SingletonBeanRegistry`的默认实现，及继承`AliasRegistry`的默认实现，增加了对`FactoryBean`（工厂`Bean`）的功能支持，
  - ConfigurableBeanFactory：提供配置`Factoroy`的各种方法，继承的`HierarchicalBeanFactory`接口对`parentFactory`提供了支持。
- ConfigurableListableBeanFactory：继承自`ListableBeanFactory`，`AutowireCapableBeanFactory`，`ConfigurableBeanFactory`，`ListableBeanFactory`提供相关方法获取`Bean`的配置信息，`AutowireCapableBeanFactory`提供创建`Bean`,自动注入、初始化及应用`Bean`的后续处理器。

现在剩下的两个主要类，`AbstractAutowireCapableBeanFactory `和 `DefaultListableBeanFactory`，前者综合了`AbstractBeanFactroy`的功能，并对`AutowireCapableBeanFactory`接口提供了实现；

后者综合了上述所有功能，提供了一个默认实现。

现在，我们来看下`XmlBeanFactory`，由于它是继承自`DefaultListableBeanFactory`，上面提供的功能它都具备，并且添加了对读取XML配置文件的支持，构造器源代码如下：

```java
public XmlBeanFactory(Resource resource) throws BeansException {
		this(resource, null);
}

public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
    // parentBeanFactory 为父类 BeanFactory，可以为空
		super(parentBeanFactory);
    // 重点关注这行代码，加载资源的真正实现
		this.reader.loadBeanDefinitions(resource);
}
```

声明`reader`属性的代码为：

```java
private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);
```

为了后续方便理解，先了解下`XmlBeanDefinitionReader`的相关类图，如下所示：

![](/java/img/XmlBeanDefinitionReader.png)

涉及到类不多，只有四个：

- `EnvironmentCapable`：定义获取`Environment`的方法；
- `BeanDefinitionReader`：定义了资源读取并转化为`BeanDefinition`的相关方法；
- `AbstractBeanDefinitionReader`：提供上面两个接口部分功能实现；
- `XmlBeanDefinitionReader`：实现从XML配置文件中加载`Bean`。

接下来，我们来分析下面这行代码到底做了什么：

```java
BeanFactory bf = new XmlBeanFactory(new ClassPathResource("MyBeanFactoryTests.xml"));
```

从代码中可以知道，`Spring` 读取 XML 文件是通过` ClassPathResource`进行封装的，在`Spring`中，对内部使用到的底层资源提供了自己的抽象结构`Resource`，提供了对资源的各种操作方法，如下所示：

```java
public interface InputStreamSource {
	InputStream getInputStream() throws IOException;
}

public interface Resource extends InputStreamSource {
	boolean exists();
    
	default boolean isReadable() {return exists();}

	default boolean isOpen() {return false;}

	default boolean isFile() {return false;}

	URL getURL() throws IOException;

	URI getURI() throws IOException;

	File getFile() throws IOException;

	default ReadableByteChannel readableChannel() throws IOException {
		return Channels.newChannel(getInputStream());
	}

	long contentLength() throws IOException;

	long lastModified() throws IOException;

	Resource createRelative(String relativePath) throws IOException;

	String getFilename();

	String getDescription();
}
```

对不同来源的资源都有对应实现，我们用到的`ClassPathResource`类也是实现了`Resource`接口，它的相关类图如下：

![](/java/img/ClassPathResource.png)

`ClassPathResource`将 Classpath下的资源返回一个`InputStream`对象，关键实现如下：

```java
public InputStream getInputStream() throws IOException {
		InputStream is;
		if (this.clazz != null) {
			is = this.clazz.getResourceAsStream(this.path);
		}
		else if (this.classLoader != null) {
			is = this.classLoader.getResourceAsStream(this.path);
		}
		else {
			is = ClassLoader.getSystemResourceAsStream(this.path);
		}
		if (is == null) {
			throw new FileNotFoundException(getDescription() + " cannot be opened because it does not exist");
		}
		return is;
}
```

其实就是通过 `class` 或 `classLoader`提供的底层方法进行调用。

## 读取配置文件

通过`Resource`相关类完成对资源的封装后，对配置文件的读取工作就交给`XmlBeanDefinitionReader`来处理了。

```java
public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
		super(parentBeanFactory);
		this.reader.loadBeanDefinitions(resource);
}
```

从上面第3行代码点进去，发现`XmlBeanDefinitionReader`首先对参数`Resource`使用`EncodedResource`进行再次封装：

```java
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(new EncodedResource(resource));
}
```

`EncodedResource` 作用是啥呢？顾名思义，大致推断它的作用是用于对资源文件进行编码使用，主要逻辑体现在`getReader`方法中，如下：

```java
public Reader getReader() throws IOException {
		if (this.charset != null) {
			return new InputStreamReader(this.resource.getInputStream(), this.charset);
		}
		else if (this.encoding != null) {
			return new InputStreamReader(this.resource.getInputStream(), this.encoding);
		}
		else {
			return new InputStreamReader(this.resource.getInputStream());
		}
}
```

从代码中可以得知，如果设置了编码属性，`Spring`会使用对应编码作为输入流的编码，返回一个`InputStreamReader`对象。

### 加载资源

接着我们来查看真正的数据准备逻辑，关键代码如下：

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    // resourcesCurrentlyBeingLoaded 是一个 ThreadLocal 变量，通过该属性来记录已经加载的资源
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
   // 判断资源是否重复加载
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}

		try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
            // InputSource 不属于 Spring,它的全路径为：org.xml.sax.InputSource,
            // 功能描述是：A single input source for an XML entity
            // 是对XML对象进行了封装
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
            // 加载Bean定义的核心逻辑
			return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
            // 移除加载的资源
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
}
```

接着来看加载`Bean`定义的核心代码，如下：

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
            // 加载 XML 文件，得到对应的 Document
			Document doc = doLoadDocument(inputSource, resource);
            // 根据返回的 Document 注册 Bean,返回注册 Bean 的数量
			int count = registerBeanDefinitions(doc, resource);
			return count;
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		// 其它异常捕获代码忽略...
	}
```

#### 获取 Document

下面我们看下`Spring`是如何从 XML 文件中获取 `Document` 的，代码如下：

```java
protected Document doLoadDocument(InputSource inputSource, Resource resource) throws Exception {
		return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
				getValidationModeForResource(resource), isNamespaceAware());
}
```

加载 `Document` 的逻辑 也是委托给 `DocumentLoader` 来处理的，看下`DocumentLoader` 的定义，只有一个方法：

```java
public interface DocumentLoader {
    
	Document loadDocument(
			InputSource inputSource, EntityResolver entityResolver,
			ErrorHandler errorHandler, int validationMode, boolean namespaceAware)
			throws Exception;
}
```

`DefaultDocumentLoader`是`DocumentLoader` 接口的实现类，获取`Document`具体逻辑如下：

```java
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
			ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

		DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
		DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
		return builder.parse(inputSource);
}
```

以上的代码算是 `SAX`解析`XML`文档的模板代码：

首先创建`DocumentBuilderFactory`，通过`DocumentBuilderFactory`创建`DocumentBuilder`，进而解析`inputSource`来返回`Document`对象。

##### 获取验证模式

`loadDocument`方法的第三个参数就是获取对 XML 文件的验证模式。

> XML 文件的验证模式保证了 XML 文件的正确性，比较常用的验证模式主要有两种：DTD （Document Type Definition，文档类型定义）和 XSD（XML Schemas Definition ,XML Schema定义描述），仅作了解，不深入展开。

验证模式的获取代码如下：

```java
protected int getValidationModeForResource(Resource resource) {
		int validationModeToUse = getValidationMode();
		// 如果手动指定验证模式,则使用指定的验证模式
    	if (validationModeToUse != VALIDATION_AUTO) {
			return validationModeToUse;
		}
    	// 如果未指定则自动检测
		int detectedMode = detectValidationMode(resource);
		if (detectedMode != VALIDATION_AUTO) {
			return detectedMode;
		}
		// 默认使用 XSD 验证模式
		return VALIDATION_XSD;
}
```

自动检测验证模式是在 `detectValidationMode` 方法中实现的，最终逻辑实现也是委托给了`XmlValidationModeDetector`来进行处理，代码如下：

```java
public int detectValidationMode(InputStream inputStream) throws IOException {
		// 在文件中查找 DOCTYPE 标识
		BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
		try {
			boolean isDtdValidated = false;
			String content;
			while ((content = reader.readLine()) != null) {
				content = consumeCommentTokens(content);
                // 如果是注释或空行则略过
				if (this.inComment || !StringUtils.hasText(content)) {
					continue;
				}
                // 如果字符串包括 DOCTYPE ，验证模式为 DTD，中断循环
				if (hasDoctype(content)) {
					isDtdValidated = true;
					break;
				}
                // 读取 < 开始符号，中断循环，因为验证模式一定会在开始符号之前
				if (hasOpeningTag(content)) {
					break;
				}
			}
			return (isDtdValidated ? VALIDATION_DTD : VALIDATION_XSD);
		}
		catch (CharConversionException ex) {
			return VALIDATION_AUTO;
		}
		finally {
			reader.close();
		}
}
```

从上面代码可以知道，`Spring` 用来检测验证模式的方法就是判断是否包含` DOCTYPE`，如果包含就是 `DTD`，没有包含就是 `XSD`。

#### 解析及注册 BeanDefinition

当把文件转换成 `Document`之后，接下来的提取及注册`Bean`就是我们要关注的重头戏了，我们顺着`registerBeanDefinitions(doc, resource)`点进去，代码实现如下：

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    // 使用 DefaultBeanDefinitionDocumentReader 实例化 BeanDefinitionDocumentReader 接口
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    // 获取加载前 BeanDefinition 数量
		int countBefore = getRegistry().getBeanDefinitionCount();
    // 加载及注册 Bean
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
	// 返回注册 Bean 的数量	
    return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

实例化`BeanDefinitionDocumentReader`的代码如下，

```java
protected BeanDefinitionDocumentReader createBeanDefinitionDocumentReader() {
		return BeanUtils.instantiateClass(this.documentReaderClass);
}
```

继续跟踪下去的话，发现也是利用反射来进行对象实例化的：

```java
public static <T> T instantiateClass(Class<T> clazz) throws BeanInstantiationException {
    // 参数校验逻辑略过...
		try {
			return instantiateClass(clazz.getDeclaredConstructor());
		}
		catch (Exception ex) {
			// ...异常处理代码略
		}
}

public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
		// 仅列出核心代码，参数校验及异常捕获略过
    // 保证给定的构造器能访问到
    ReflectionUtils.makeAccessible(ctor);
    // 获取构造器参数类型
    Class<?>[] parameterTypes = ctor.getParameterTypes();
	Object[] argsWithDefaultValues = new Object[args.length];
	for (int i = 0 ; i < args.length; i++) {
		if (args[i] == null) {
            // 如果参数为空，从 DEFAULT_TYPE_VALUES 获取对应类型的默认值，找不到参数值默认给 null
           // DEFAULT_TYPE_VALUES 是个不可修改的map，对以下类型指定了默认值:
           // boolean -> false
           // byte,short,int,long -> 0 
			Class<?> parameterType = parameterTypes[i];
			argsWithDefaultValues[i] = (parameterType.isPrimitive() ? DEFAULT_TYPE_VALUES.get(parameterType) : null);
		}
		else {
            // 设置传过来的参数
			argsWithDefaultValues[i] = args[i];
		}
	}
    // 调用 JDK 方法，利用反射进行实例化
	return ctor.newInstance(argsWithDefaultValues);
}
```

`getRegistry().getBeanDefinitionCount()` 获取当前已经注册的 `Bean` 的数量，具体逻辑如下：

```java
// 返回当前类，即 XmlBeanDefinitionReader 持有的 BeanDefinitionRegistry
public final BeanDefinitionRegistry getRegistry() {
		return this.registry;
}
// XmlBeanDefinitionReader 是在哪里初始化的呢？加载 XmlBeanFactory 类的时候就对其进行了初始化
private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);
// 回头看上面 XmlBeanFactory 的相关类图，DefaultListableBeanFactory 实现了 BeanDefinitionRegistry 接口 ,即 getRegistry().getBeanDefinitionCount() 调用的方法即是 DefaultListableBeanFactory 的实现，代码如下：
public int getBeanDefinitionCount() {
	return this.beanDefinitionMap.size();
}
// 注册的 Bean 缓存在 beanDefinitionMap 中，声明如下：
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
```

现在就到了加载注册 `Bean` 的核心代码了：`documentReader.registerBeanDefinitions(doc, createReaderContext(resource))`，点进去实现如下：

```java
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		doRegisterBeanDefinitions(doc.getDocumentElement());
}

protected void doRegisterBeanDefinitions(Element root) {
    // 初始化 BeanDefinition 解析类
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);
  		// 如果是默认命名空间：通过与 Spring 中固定的命名空间（http://www.springframework.org/schema/beans）进行比对来判断
		if (this.delegate.isDefaultNamespace(root)) {
            // 处理 profile 属性
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				// 设置 profile 值
               if(!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					return;
				}
			}
		}
		// 解析前处理，留给子类实现
		preProcessXml(root);
    	// 解析 BeanDefinition 核心逻辑
		parseBeanDefinitions(root, this.delegate);
    	// 解析后处理，留给子类实现
		postProcessXml(root);

		this.delegate = parent;
	}
```

继续从这行 `parseBeanDefinitions(root, this.delegate)` 代码点进去：

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
	// 针对使用默认标签的 Bean 处理
    if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
                    // 再次对命名空间进行判断
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
            // 针对 自定义标签 Bean 的处理
			delegate.parseCustomElement(root);
		}
}
```

##### 默认标签处理

可以看到，`Spring` 针对默认标签和自定义标签走了不同的逻辑，下面先看默认标签的解析，从`parseDefaultElement(ele, delegate)` 点进去，代码如下：

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		// import 标签处理	
    	if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
    	// alias 标签处理
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
    	// bean 标签处理
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
    	// beans 标签，又调回 doRegisterBeanDefinitions 方法
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
```

针对上面四种标签的处理，`bean` 标签的解析最复杂也最重要，所以以该标签的处理逻辑来着重分析，由于后续篇幅过长，在下篇文章中来分析 `bean` 标签的解析过程。



